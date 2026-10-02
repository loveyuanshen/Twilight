---
title: 在Mac本地部署的折腾历程
published: 2026-09-30T03:05:00.000+08:00
updated: 2026-10-03T03:05:00.000+08:00
description: Mac跑本地模型的折腾历程
cover: d90d7c59e85ddce493775a1b601de8e5dc0d7139.jpg
tags:
  - LLM
category: 本地部署LLM
draft: false
---

# 在Mac本地部署的折腾历程



## 前言

笔者没买这台Mac之前,在Android端部署过一个huggging face社区的蒸馏模型,2B参数

问简单问题能回答,上下文一旦超过1000,回答质量便会明显下降,最终端侧模型的方案不了了之.

买了这台M·ac后,闲来无事便想折腾试试.

笔者的配置:

Mac book Air标配 ,M5处理器.配备10+8核处理器和神经网络加速器.





## 初次尝试



### 准备工作

在实际部署之前,我给Rikkanhub加载了一个hugging face的MCP,用于检索模型参数.

让Qwen 3.8帮忙检索M5标配的Mac大概能跑得动多大参数的LLM,给的回答是9b到12b之间,12b以上即使跑起来也会动用swap,降低可用性.

接着我让它检索适合本地跑AI的客户端.

它给我两种主流框架:ollama和LM studio.

其中ollama配置简单,适合小白使用,vllm,omlx更适合量化和集群部署,并不适合个人developer自行部署.

首先我尝试了olllama.通过home brew 安装.



```bash
Brew install ollama
```



### 拉取模型并创建配置

安装ollama后,打开终端并输入ollama启动服务.



> [!NOTE]
>
> 关于模型的选择:Qwen最佳,Gemma其次,其他的如GPT OSS系列没试过,不作评价.我最终选用了Qwen3.5 9B作为基座模型.本文后续将以该模型为主要示例讲解.
>
> 个人观点:模型优先根据自己的需求选择,不需要tool_call尽量选择蒸馏模型,垂直领域性能更好.

输入

```bash
ollama pull qwen3.5:9b
```

拉取模型.

等待进度跑满,模型下载完成.



### 限制KV cache 



- 什么是KV cache?

  **KV Cache 是大语言模型推理中用于缓存 Key（键）和值（Value）向量的技术，用于避免重复计算，提高生成速度和效率。**

* 为何限制KV cache?

  KV cache 会随模型运行而占用内存.是模型推理必需的,我们不希望模型刚刚跑起来就让它吃掉过量内存,因此我们会根据需要限制KV cache ,节省资源.
* 如何限制KV cache?

在o l l ma中,可通过创建配置文件实现.



```bash
mkdir -p ~/models && cat > ~/models/Modelfile <<'EOF'
FROM qwen3.5:9b
PARAMETER num_ctx 8192
PARAMETER temperature 0.65
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.1
SYSTEM 你是中文文本编辑。保持原意和事实，不新增内容；改得自然、清楚，适合博客文章的写作。
EOF
```

其中的 **PARAMETER num_ctx 8192**就是设置的参数,用以限制KV cache

然后



### 创建配置,启动模型并测试回答



```bash
ollama create qwen-polish -f ~/models/Modelfile
ollama run qwen-polish
```

稍事等待,在>>>后输入想要提问的内容.

观察性能监视器,如果内存占用升高,CPU温度明显升高,说明模型已经成功挂载.

模型默认会开启思考,看到回答出现即可,按下**Ctrl+C**即可终止对话.

输入

```bash
/bye
```

即可离开对话,输入



```bash
ollama stop qwen-polish
```

即可将模型从内存卸载.



### API本机调用

Ollama 运行会自动开放API.按表填写即可

| 项目       | 填写内容                        |
| -------- | --------------------------- |
| Base URL | `http://localhost:11434/v1` |
| 模型 ID    | `qwen-polish`               |
| API Key  | `ollama`（任意非空字符串）           |
| 聊天路径     | `POST /v1/chat/completions` |

### 发现的问题



- 模型默认思考导致回答截断

  经过排查,因为模型默认思考后回答,所以每次都会消耗很多时间.CPU也飙到了93度

  在高挑战任务中这无疑是好的,但是思维链也会占用上下文,导致回答常常截断.

  解决方法:每次跑模型时,预先输入/set nothink命令以限制思考.或者启动时加入参数:



```bash
ollama run qwen-polish --think=false
```



- 推理内存占用过高

  回答问题时功耗能干到34w,这可是一台Mac book啊.

  排查发现运行了f16 缓存和Q8量化.

  怪不得.

  在修改了环境变量后,内存占用恢复正常



```bash
launchctl setenv OLLAMA_KV_CACHE_TYPE q4_0
```



### ollama 常用命令

| 命令             | 作用                 |
| -------------- | ------------------ |
| `/bye`         | 退出对话，回到终端          |
| `/clear`       | 清空当前聊天上下文，模型仍保持加载  |
| `/load 名称`     | 加载已保存的模型或会话        |
| `/save 名称`     | 把当前参数和系统提示保存成一个新模型 |
| `/?` 或 `/help` | 查看命令帮助             |
| `/? shortcuts` | 查看当前版本的快捷键         |

| 命令                 | 作用         |
| ------------------ | ---------- |
| `/set nothink`     | 关闭思考，润色时使用 |
| `/set think`       | 重新开启思考     |
| `/set verbose`     | 显示速度、耗时等统计 |
| `/set quiet`       | 隐藏这些统计     |
| `/set format json` | 要求输出 JSON  |
| `/set noformat`    | 取消 JSON 模式 |
| `/set history`     | 记录命令历史     |
| `/set nohistory`   | 不记录命令历史    |
| `/set wordwrap`    | 按窗口宽度换行    |
| `/set nowordwrap`  | 关闭自动换行     |

### 常见变量参考

| 参数               | 含义                      |
| ---------------- | ----------------------- |
| `num_ctx`        | 上下文长度，16GB 上建议 **8192** |
| `temperature`    | 随机性，润色用 **0.6～0.7**     |
| `top_p`          | 采样范围，常用 **0.9**         |
| `repeat_penalty` | 重复惩罚，常用 **1.1**         |



### 变量设置



> /set parameter num_ctx 8192
>
> /set parameter temperature 0.65
>
> /set parameter top_p 0.9
>
> /set parameter repeat_penalty 1.1



## 转向MLX框架



### 起因

前文跑的模型格式为.guff,是llma.cpp 的推理框架,无法使用Apple的神经网络加速.

为了进一步提升模型性能,我放弃了现有的ollama,转而使用Apple独有的MLX框架,同时使用量化模型Jackrong/MLX-Qwen3.5-9B-Claude-4.6-Opus-Reasoning-Distilled-v2-4bit以提高回答质量.

**关于MLX**

MLX 是苹果公司于 2023 年 12 月推出的开源机器学习阵列框架，专为 Apple Silicon（M 系列芯片） 深度优化。你可以把它理解为 “苹果生态中的 PyTorch”——一个充分利用苹果芯片统一内存架构和 Metal GPU 加速能力，用于训练和部署 AI 模型的原生框架。

### 选择推理框架

主流的MLX框架推理应用有:LM Studio 和o MLX.

oMLX适用于集群部署,LM Studio是GUI应用,占用内存更高,因此二者都不能用.

Qwen 建议我使用MLX_lm.理由是它可以直接调用底层m l x库,没有性能损耗.





### 安装并启用mlx-lm



```bash
# 推荐用 uv（比 pip 快 10-100 倍）
brew install uv
uv venv ~/.venv/mlx-lm
source ~/.venv/mlx-lm/bin/activate
uv pip install mlx-lm

# 或者传统方式
python3 -m venv ~/.venv/mlx-lm
source ~/.venv/mlx-lm/bin/activate
pip install mlx-lm
```

任选其一即可





### 拉取模型





```bash
# 安装 huggingface-cli（如果没有）
pip install huggingface_hub

# 下载蒸馏版主模型
huggingface-cli download Jackrong/MLX-Qwen3.5-9B-Claude-4.6-Opus-Reasoning-Distilled-v2-4bit \
  --local-dir ~/models/qwen35-opus-v2-4bit

# 下载 MTP 草稿模型（加速用）
huggingface-cli download mlx-community/Qwen3.5-9B-MTP-4bit \
  --local-dir ~/models/qwen35-mtp-4bit
```



> 如果下载速度慢，可以设置镜像：在下载命令前加一行
>
> &#x20; export HF_ENDPOINT=https\://hf-mirror.com&#x20;



### 运行模型并测试



```bash
mlx_lm.chat --model ~/models/qwen35-opus-v2-4bit
```

等待几秒,输入指令测试.



```bash
# 前台运行（看日志，Ctrl+C 停止）
mlx_lm.server \
  --model ~/models/qwen35-opus-v2-4bit \
  --draft-model ~/models/qwen35-mtp-4bit \
  --port 8080

# 后台运行（关终端不停）
nohup mlx_lm.server \
  --model ~/models/qwen35-opus-v2-4bit \
  --draft-model ~/models/qwen35-mtp-4bit \
  --port 8080 \
  > /tmp/mlx-server.log 2>&1 &
```



```
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"你好"}],"max_tokens":64}'
```

查看内存占用

```
du -sh ~/models/*
# 示例输出：
# 5.3G  /Users/你的用户名/models/qwen35-opus-v2-4bit
# 1.1G  /Users/你的用户名/models/qwen35-mtp-4bit
```

参数/命令速览表

| 参数                     | 说明                |           |             |
| ---------------------- | ----------------- | :-------- | :---------- |
| `--model`              | 模型路径              |           |             |
| `--temp`               | 温度                |           |             |
| `--top-p`              | 核采样               |           |             |
| `--max-tokens`         | 单次最大输出            |           |             |
| `--system-prompt`      | 系统提示词             |           |             |
| 参数                     | 说明                | 默认值       |             |
| ---                    | ---               | ---       |             |
| `--model`              | 主模型路径             | 必填        |             |
| `--draft-model`        | MTP 草稿模型          | 无         |             |
| `--port`               | 监听端口              | 8080      |             |
| `--host`               | 监听地址              | 127.0.0.1 |             |
| `--temp`               | 默认温度              | 1.0       |             |
| `--top-p`              | 默认 top_p         | 1.0       |             |
| `--max-tokens`         | 默认最大生成            | 100       |             |
| `--prompt-cache-size`  | prompt 缓存 token 数 | 512       |             |
| 参数                     | 说明                | 默认值       | 写作推荐        |
| ---                    | ---               | ---       | ---         |
| `--model`              | 模型路径或 HF 名        | 必填        | 本地路径        |
| `--prompt`             | 输入文本              | 必填        | —           |
| `--max-tokens`         | 最大生成 token 数      | 100       | 2048-4096   |
| `--temp`               | 温度（创意度）           | 1.0       | 0.7（润色 0.5） |
| `--top-p`              | 核采样               | 1.0       | 0.8         |
| `--top-k`              | Top-K 采样          | 0         | 20          |
| `--repetition-penalty` | 重复惩罚              | 无         | 1.05-1.1    |
| `--seed`               | 随机种子              | 无         | 固定值（可复现）    |
| 命令                     | 作用                |           |             |
| ---                    | ---               |           |             |
| `/quit`                | 退出                |           |             |
| `/clear`               | 清空对话历史            |           |             |
| `/system 你是写作助手`       | 设置系统提示词           |           |             |



### 通过API调用

| 参数                    | 值                                               |
| --------------------- | ----------------------------------------------- |
| **API 地址 / Base URL** | `http://127.0.0.1:8080/v1`                      |
| **API Key**           | 随意填（如 `none`、`sk-local`、`1234`），mlx-lm 不校验      |
| **模型名称**              | 随意填（如 `default`、`local`、`qwen`），服务器不校验          |
| **协议**                | OpenAI Chat Completions（`/v1/chat/completions`） |



## 最终结果

测试结果:21.19 tokens/s  →22.4 tokens/s。(未使用MTP)

温度93度→45度

内存占用:下降0.8G→1.2G



