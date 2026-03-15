+++
date = '2026-03-14T12:06:42+08:00'
draft = false
title = '在MacOS上基于docker安装OpenClaw-使用本地部署LLM'
+++

# 导语
近期小龙虾爆火，整个社会陷入了疯狂，有兴奋、有焦虑，加上自媒体疯狂宣传，让人觉得小龙虾已经无所不能，再不掌握就要给这个世界抛弃了。秉着实事求是的精神，我决定先手戳一个纯本地部署的小龙虾，毕竟打破盲目崇拜的最有效办法就是掌握它，而且还是零成本（当然个人电脑和宽带还是要有的哈哈）。

# 环境说明
- Macbook Pro (Apple M4 Pro)

整体思路是，考虑小龙虾的安全性和token消耗问题。我计划把小龙虾部署到本地docker环境中，同时在mac上安装ollama跑一个本地大模型，docker中的小龙虾访问宿主记得本地模型，从而解决以上两个问题。不过当然会比较烧cpu，而且本地模型性能会不如大模型厂商提供的服务，但是用于学习探索也足够了。

# 安装步骤
## ollama
MacOS上只需要执行一下命令安装即可，安装后右下角选择想要的大模型，然后随便发个消息，系统就会自动下载模型。
```shell
brew install ollama
```
图形界面需要访问`http://ollama.com`安装
![ollama图形界面](/images/openclaw_ollama_index.png)

## qwen3-coder:30B
我本地下载的是qwen3-coder:30B
![ollama下载大模型](/images/openclaw_ollama_down_model.png)

## 测试本地模型API是否可用
ollama除了提供图形窗口界面，还默认监听了本地11434端口，可通过浏览器访问如下地址验证API是否可用。
```shell
http://localhost:11434/v1/models
```
如果正常，会有如下图的模型信息输出。
![ollama API输出](/images/openclaw_ollama_api.png)

## docker
这里不使用dockerDesktop，而使用替代品colima，主要是轻量/免费，当然缺点是只有命令行。
```shell
brew install colima docker docker-compose
```
安装后执行`colima start`后，docker正常输出说面安装成功。

## 克隆OpenClaw代码和拉起容器
```
git clone https://github.com/OpenClaw/OpenClaw.git
```
这里先改一下docker-compose.yaml，改成从源码编译镜像，主打全手动自主撸小龙虾。
```yaml
diff --git a/docker-compose.yml b/docker-compose.yml
index 614a1f8d5..6afad2cb0 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -1,6 +1,13 @@
 services:
   openclaw-gateway:
-    image: ${OPENCLAW_IMAGE:-openclaw:local}
+    #image: ${OPENCLAW_IMAGE:-openclaw:local}
+    build:
+      context: .
+      dockerfile: Dockerfile
+    deploy:
+      resources:
+        limits:
+          memory: 4g
     environment:
       HOME: /home/node
       TERM: xterm-256color
@@ -25,10 +32,18 @@ services:
         "${OPENCLAW_GATEWAY_BIND:-lan}",
         "--port",
         "18789",
+        "--allow-unconfigured",
       ]

   openclaw-cli:
-    image: ${OPENCLAW_IMAGE:-openclaw:local}
+    #image: ${OPENCLAW_IMAGE:-openclaw:local}
+    build:
+      context: .
+      dockerfile: Dockerfile
+    deploy:
+      resources:
+        limits:
+          memory: 4g
     environment:
       HOME: /home/node
       TERM: xterm-256color
```

不过编译时，由于colima默认资源的限制，把Dockerfile也调整下，避免node.js编译时OOM
```yaml
diff --git a/Dockerfile b/Dockerfile
index 255340cb0..b1cf25826 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -25,6 +25,9 @@ COPY --chown=node:node scripts ./scripts
 USER node
 RUN pnpm install --frozen-lockfile

+ENV NODE_OPTIONS="--max-old-space-size=3072"
+ENV MAX_JOBS=2
+
 # Optionally install Chromium and Xvfb for browser automation.
 # Build with: docker build --build-arg OPENCLAW_INSTALL_BROWSER=1 ...
 # Adds ~300MB but eliminates the 60-90s Playwright install on every container start.
```

## 测试docker中访问宿主机大模型联通性
```shell
docker-compose up -d
```
执行以上命令拉起容器，进入容器执行以下命令，如果有输出模型信息即可。
```shell
curl http://host.docker.internal:11434/v1/models
```

## 修改配置和环境变量
- 环境变量配置
项目根目录下执行
```shell
cp .env.example .env
```
然后.env文件增加一下配置，安全的关键之一在于OPENCLAW_GATEWAY_PORT和OPENCLAW_BRIDGE_PORT要配置成127.0.0.1本地回环IP，避免把自己暴露到公网的可能性。
```
OPENCLAW_GATEWAY_TOKEN=xxxxxx #自己随便配置一个字符串就行
OPENCLAW_CONFIG_DIR=../config
OPENCLAW_WORKSPACE_DIR=../workspace
OPENCLAW_GATEWAY_PORT=127.0.0.1:18789
OPENCLAW_BRIDGE_PORT=127.0.0.1:18790
```
- 项目配置文件
在上面OPENCLAW_CONFIG_DIR的目录(../config)中，增加配置文件openclaw.json，内容如下。
注意gateway.auth.token即上面OPENCLAW_GATEWAY_TOKEN的值。
```json
{
  "gateway": {
    "auth": {
      "token": "xxxxxx"
    },
    "controlUi": {
      "allowedOrigins": [
        "http://localhost:18789",
        "http://127.0.0.1:18789"
      ]
    }
  },
  "models": {
    "providers": {
      "ollama-local": {
        "baseUrl": "http://host.docker.internal:11434/v1",
        "apiKey": "ollama",
        "api": "openai-completions",
        "models": [
          {
            "id": "qwen3-coder:30b",
            "name": "Qwen 3 30B",
            "contextWindow": 32768,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "ollama-local/qwen3-coder:30b"
    }
  }
}
```
## 启动效果
```shell
docker-compose up -d
```
## 连接QQ机器人
在`https://q.qq.com/qqbot/openclaw/index.html`页面创建一个qq机器人，进入容器按页面介绍执行命令接入机器人即可。
这里遇到点小插曲，容器里找不到openclaw命令行，问了AI，可以用node /app/dist/index.js 来代替openclaw命令。

# 效果
![网关首页](/images/openclaw_index_healthy.png)
![QQ聊天机器人界面](/images/openclaw_qq_chatbot.png)
