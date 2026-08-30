# DeepTutor 与现有 DeepSeek 调度平台联动

这套部署采用“两个服务独立运行、通过 API 联动”的方式，避免把 DeepTutor 代码直接塞进 DeepSeek Harness 主程序。

## 目标架构

```text
手机 / 电脑 / 微信
        |
        v
现有 DeepSeek 调度平台
        |
        +--> 普通 AI / Agent 任务
        |
        +--> 学习入口 / DeepTutor
                    |
                    +--> 学习空间 / 知识库 / 记忆
                    |
                    +--> OpenAI-compatible API
                              |
                              v
                    现有 DeepSeek 调度网关
```

DeepTutor 自己保存学习空间、知识库、记忆和设置；模型调用优先走现有调度平台。如果调度平台暂时不可用，DeepTutor 仍可单独改用其他模型提供方，不影响学习数据。

## 1. 启动 DeepTutor

在服务器上进入本目录：

```bash
cp .env.example .env
docker compose pull
docker compose up -d
```

查看状态：

```bash
docker compose ps
docker logs --tail 100 deeptutor
```

默认只监听服务器本机：

```text
http://127.0.0.1:3782
```

这样不会把 3782 端口直接暴露到公网。

## 2. 连接现有 DeepSeek 调度平台

DeepTutor 支持自定义 OpenAI-compatible provider。由于 DeepTutor 运行在 Docker 容器中，同机访问宿主机服务使用：

```text
http://host.docker.internal:3080/v1
```

建议在 DeepTutor 的 Settings / Models 中新建 Custom 或 OpenAI-compatible provider：

```text
Base URL: http://host.docker.internal:3080/v1
Model:    deepseek-chat
API Key:  使用你现有调度网关要求的 token
```

注意：这里要求现有调度平台真正提供 OpenAI Chat Completions 兼容接口，例如：

```text
GET  /v1/models
POST /v1/chat/completions
```

如果当前 3080 只有 DeepSeek Harness Web UI，而没有上述 API，就需要在调度平台前面保留/补上已有的 API Gateway 适配层；不要把 DeepTutor 直接指向 Web UI 页面。

## 3. 知识库需要 Embedding

DeepTutor 的知识库/RAG 除了聊天模型，还需要 embedding 模型。推荐两种方式：

1. 如果现有调度平台已经代理 embedding：让 DeepTutor 也指向同一个网关。
2. 如果当前调度平台只代理聊天模型：先单独给 DeepTutor 配一个支持 embeddings 的 provider。

不要为了“统一入口”而让知识库功能失效。

## 4. Cloudflare 域名

推荐给 DeepTutor 单独一个子域名，例如：

```text
study.zxzxzx.de
```

Cloudflare Tunnel 的 service 指向：

```text
http://localhost:3782
```

保留原来的 AI 总入口不变，例如：

```text
deai.zxzxzx.de  -> 现有 DeepSeek 调度平台
study.zxzxzx.de -> DeepTutor
```

这样两个前端独立，后台模型调用仍然可以统一走 DeepSeek 调度网关。

## 5. 最终路由建议

后续在现有调度平台增加“学习任务”路由：

- 普通问答：继续走原来的 DeepSeek / GPT / Claude 调度。
- 编程任务：继续走 Harness / Codex 等执行器。
- 学习任务：跳转或调用 DeepTutor；DeepTutor 再通过统一模型网关请求模型。

这样 DeepTutor 更新失败不会拖垮主调度平台，主平台升级也不会破坏 DeepTutor 的学习记录。

## 6. 数据与安全

DeepTutor 数据保存在 Docker volume：

```text
deeptutor-data
```

不要把真实 API Key 写进仓库。公网访问建议继续走 Cloudflare Tunnel / Access，不要直接开放 3782 或 DeepTutor 后端端口。
