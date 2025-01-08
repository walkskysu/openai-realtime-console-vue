### 项目介绍

这个项目是基于 [openai-realtime-console](https://github.com/openai/openai-realtime-console/tree/websockets) 修改而来的，原始项目使用了 React 架构，而本项目将其架构转换为 Vite + Vue3。

---

### 快速开始

#### 1. 克隆整个项目并使用 npm 启动

```bash
git clone https://github.com/walkskysu/openai-realtime-console-vue.git

cd openai-realtime-console-vue

# npm
npm install
npm run dev
```

首次启动时，会要求输入 OpenAI 的 API 密钥。该模式通过客户端直接连接 OpenAI，示意图如下：

Client（浏览器） <------> OpenAI API Server

---

#### 2. 使用 Relay Server 启动

Relay Server 模式不同于客户端直连模式，它通过 Relay Server 中转与 OpenAI API 通信。示意图如下：

Client（浏览器） <------> Relay Server <------> OpenAI API Server

启动 Relay Server 之前，需要先修改 `.env` 文件：

```conf
OPENAI_API_KEY=YOUR_API_KEY
VITE_APP_LOCAL_RELAY_SERVER_URL=http://localhost:8081
```

然后启动 Relay Server：

```shell
npm run relay
```

Relay Server 将会在 `localhost:8081` 启动。

**注意**：如果客户端和 Relay Server 部署在不同的机器上，需要在客户端配置 `.env` 文件，增加以下内容：

```conf
VITE_APP_LOCAL_RELAY_SERVER_URL=http://localhost:8081
```

---

### 更多功能说明

更多功能说明，请参考原始项目的文档介绍：  
[https://github.com/openai/openai-realtime-console/tree/websockets](https://github.com/openai/openai-realtime-console/tree/websockets)
