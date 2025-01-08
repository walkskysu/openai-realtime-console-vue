English | [中文](README.zh.md)

### Introduction

This project is based on modifications of [openai-realtime-console](https://github.com/openai/openai-realtime-console/tree/websockets). The original project was built with a React architecture, while this project has migrated the architecture to Vite + Vue3.

---

### Getting Started

#### 1. Clone the project and run it using npm

```bash
git clone https://github.com/walkskysu/openai-realtime-console-vue.git

cd openai-realtime-console-vue

# npm
npm install
npm run dev
```

On the first launch, you will be prompted to enter your OpenAI API key. This mode connects directly to the OpenAI server. The architecture is as follows:

Client (Browser) <------> OpenAI API Server

---

#### 2. Run with a Relay Server

The Relay Server mode is different from the direct connection mode, as it uses a Relay Server to act as an intermediary between the client and the OpenAI API. The architecture is as follows:

Client (Browser) <------> Relay Server <------> OpenAI API Server

Before starting the Relay Server, you need to modify the `.env` file:

```conf
OPENAI_API_KEY=YOUR_API_KEY
VITE_APP_LOCAL_RELAY_SERVER_URL=http://localhost:8081
```

Then, start the Relay Server:

```shell
npm run relay
```

The Relay Server will run on `localhost:8081`.

**Note**: If the client and the Relay Server are deployed on different machines, you need to add the following configuration to the `.env` file on the client side:

```conf
VITE_APP_LOCAL_RELAY_SERVER_URL=http://localhost:8081
```

---

### Further Information

For more information about features, please refer to the original project documentation:  
[https://github.com/openai/openai-realtime-console/tree/websockets](https://github.com/openai/openai-realtime-console/tree/websockets)
