<h1 align="center">4gbVramStrata</h1>

<p align="center"><b>在配备 4 GB 显卡的笔记本电脑上运行 24B 到 40B 的混合专家模型。</b><br>
无需云端，无需 API 密钥，所有数据都不离开本机。</p>

<p align="center">
  <a href="#measured-on-a-4-gb-laptop">实测</a> ·
  <a href="#the-models">模型</a> ·
  <a href="#install">安装</a> ·
  <a href="#use-it">使用</a> ·
  <a href="#how-it-works">工作原理</a> ·
  <a href="docs/BENCH-4GB.md">基准测试</a>
</p>

---

一个 35B 的混合专家模型拥有约 350 亿个参数，但处理每个 token 时只使用其中约 30 亿个。其余
参数保持不动。这正是本项目利用的切入点：将整个模型保留在电脑内存中，将下一个 token 所需的少数专家
保留在显卡中，让 CPU 计算无法容纳的部分。4 GB 的显卡就足够了，
因为它永远不需要容纳整个模型——只需要容纳其中最热的那一部分。

本项目是 [PolyStrata](https://github.com/VecSzn/PolyStrata) 的分支，而 PolyStrata 又是
[Strata](https://github.com/Niko1221/Strata) 的分支。它在一台机器上进行了调优、测量和文档记录：一台 **RTX 3050
笔记本，4 GB 显存**，Ryzen 5 5500U 和 32 GB 内存，运行 Windows 11。

## 在 4 GB 笔记本上的实测

参考机器：RTX 3050 Laptop（4 GB，`sm_86`），Ryzen 5 5500U（6 核 12 线程），32 GB DDR4-3200，Windows 11。
引擎从本仓库构建，使用 CUDA 13，支持 `sm_75` 到 `sm_120`。**向服务器发送聊天请求——即
用户实际使用的场景**——使用约 1,900 个 token 的已保存提示词，以一个问题结尾，生成 128 个新 token，32K 上下文，
在安静的机器上运行八次。取中位数及范围：

| 模型 | 文件 | 读取提示词 | 生成回答 | 显存中的专家 |
| --- | ---: | ---: | ---: | ---: |
| Clover-35B-A3B-Preview APEX-I-NANOPLUS | 11.7 GB | 449 tokens/s | **38.1 tokens/s**（32.6-39.1） | 10,240 中的 529 个 |
| Flint 40B-A2.5B UD-Q3_K_XL | 17.2 GB | 331 tokens/s | **21.9 tokens/s**（20.5-22.4） | 12,032 中的 518 个 |

提示词为 `bench/speed/prompt.txt`，测试工具为 `bench/speed/run.py`，每次运行都会向
`bench/speed/runs.log` 追加一行记录，包含引擎的 SHA-256。同一提示词在引擎的原始 token-id 模式下——模型
续写文本而非回答问题——写入速度为 31.1 和 21.6 tokens/s：这是不同的任务，不是回答速度。

设置项为安装程序写入的配置：`--tail-skip 6`、`--no-graphs`，对于 Clover 在 6 GB 或以下的显卡上使用 `--kv q8_0`；
对于 Flint 使用 `--prompt-batch 512`、`--expert-swaps 32` 和 `--no-graphs`。

**本表早期版本曾显示 47.5 和 40.7 tokens/s，但这一数据无法复现。** 其提示词未保存，
日志已丢失——本页面现已更正这一错误——同样的请求如今写入速度为 38.1 和 21.9。旧数据保留在
[docs/BENCH-4GB.md](docs/BENCH-4GB.md) 中，标记为不可复现，因为抹去它会掩盖曾经发生的事。

两个分别为 12 GB 和 17 GB 的文件在新提示词上以每秒 22 到 38 个 token 的速度回答，尽管它们都完全无法放入显卡：
加载约需两秒，之后显卡运行其持有的专家，CPU 运行其余部分。
增长到 32K 的对话速度大约减半（[曲线图](docs/BENCH-4GB.md#context-and-speed)）；使用 8 位
注意力缓存时损失更小——在 24,000 token 上下文的原始续写任务中，39.0 对 30.9
tokens/s，相同提示词和设置，三轮测试。

每个数字、其日志、逐项测量的设置、对话增长到 32K 时的开销，以及**未**测量的内容：
**[docs/BENCH-4GB.md](docs/BENCH-4GB.md)**。

## 模型

十二个模型，每个来自厂商自己的 GGUF 或知名量化者的版本。其中两个在此 4 GB 机器上进行了实测；其余
列出的是安装程序对此显卡的估算值。大小为最小下载量；详细信息、量化大小、
厂商评分和来源：**[docs/MODELS-4GB.md](docs/MODELS-4GB.md)**。

| 模型 | 参数 | 最小文件 | 在 4 GB 显卡上 |
| --- | --- | ---: | --- |
| **Clover-35B-A3B-Preview**（DiffuSpace） | 35B / 3B 激活 | 13 GB | **已实测**，38.1 tokens/s |
| **Flint 40B-A2.5B**（Recursive） | 40B / 2.5B 激活 | 18 GB | **已实测**，21.9 tokens/s |
| **LFM2.5-24B-A2B**（Liquid AI） | 24B / 2B 激活 | 11 GB | 应该能启动；此处未运行 |
| **Qwen3.6-35B-A3B**（Qwen） | 35B / 3B 激活 | 13 GB | 不能：任何专家之前已占 4.2 GB |
| **Ornith 1.5-35B-A3B**（ornith-ai） | 35B / 3B 激活 | 14 GB | 最小尺寸应该能启动；此处未运行 |
| **Tiel Coder 35B-A3B**（peculiar-ragdoll） | 35B / 3B 激活 | 14 GB | 不能：任何专家之前已占 4.4 GB |
| **Gemma 4 26B-A4B**（Google） | 26B / 3.8B 激活 | 11 GB | 不能：任何专家之前已占 4.1 GB |
| **MiniCPM-V 4.7 35B-A3B**（OpenBMB） | 35B / 3B 激活 | 16 GB | 应该能启动；此处未运行 |
| **Kolibri 1**（Aleph Alpha） | 78B / 3.5B 激活 | 29 GB | 不能：任何专家之前已占 4.1 GB |
| **GLM 5.3 Flash**（Zhipu） | 320B / 18B 激活 | 79 GB | 不能：需要 32 GB 显卡 |
| **DeepSeek V4 Flash**（DeepSeek） | 284B / 13B 激活 | 91 GB | 不能：需要 32 GB 显卡 |
| **Qwen3.8-Flash-Next**（Qwen） | 125B / 6B + 51B n-gram | 58 GB | 不能：需要 12 GB 显卡和 32 GB 内存 |

"任何专家之前"指的是某个尺寸在显存中占用的稠密权重、注意力缓存和缓冲区。剩余显存足够的显卡
可以启动，但专家缓存更小；**[docs/BENCH-4GB.md](docs/BENCH-4GB.md#the-engine-this-release-ships)**
说明了其代价。

## 安装

Windows 10 或 11，64 位。NVIDIA RTX 20 系列或更新显卡，驱动 580 或更新。16 GB 内存可运行较小
模型，32 GB 可运行上述两个实测模型。Python 3.10 或更新：如果缺失，安装程序会为你的用户账户安装，
无需管理员权限。

1. 下载 **[最新版本](https://github.com/RandoTeam/4gbVramStrata/releases/latest)** 并解压到任意
   有足够空间存放所需模型的位置（11 到 92 GB）。
2. 运行 `START-HERE.bat`。首次运行时它会检查电脑，提供适合的模型选项，下载一个，写入运行
   配置并启动服务器。
3. 后续运行会启动已安装的内容。`SETUP.bat` 会再次询问——另一个模型、另一个尺寸、另一个上下文。
   安装程序会为每个已安装模型写入一个 `run-<model>.bat`：一键启动，无需回答任何问题。

首次运行会下载 Python 包（几百 MB）和模型。无需编译器：引擎
已包含在发行版中，为所有 RTX 20 到 RTX 50 显卡构建。

## 使用

服务器同时支持两种 API 协议，因此大多数工具只需更改一个地址：

| | |
| --- | --- |
| OpenAI | `http://127.0.0.1:8080/v1` — `/chat/completions`、`/completions`、`/models`、工具调用、流式传输 |
| Anthropic | `http://127.0.0.1:8080/v1/messages` — Claude Code 使用的接口 |
| Web 应用 | `http://127.0.0.1:8080` — 聊天、模型切换、实时监控 |
| MCP | `docs/MCP_SERVER.md` — 从 AI 助手安装、启动、停止和切换 |

任何 API 密钥均可（本地服务器不检查）。将代理指向该地址，它就会像托管模型一样工作，
唯一的区别是：长提示词的第一个 token 较慢，而且它就在你的桌面上运行。

## 工作原理

使本项目成为可能的引擎（`strata-fourgbvram`，位于 `src/fourgbvram/`）就地读取模型的 GGUF 文件。
稠密权重、注意力缓存和每个缓存专家一个槽位驻留在显存中；其他所有专家留在
文件中，由操作系统缓存在内存中。每个 token 在每层路由到少数几个专家。引擎将
它不持有的专家复制到它使用最少的专家的槽位中，通过 PCIe 传输，在回答生成的同时进行——
这次复制就是小显卡的代价，本项目的大部分工作就是让它更便宜，并将它花在
正确的专家上。

第二个引擎 `strata`（`src/`、`include/`）运行 Qwen3.8-Flash-Next，原样来自 Strata。

## 出问题了？

- **显卡似乎是问题所在。** 另一个程序正在使用显存：关闭带硬件加速的浏览器、
  其他模型服务器、任何带 GPU 视图的程序。4 GB 上一次只能运行一个模型。
- **回答比上表慢得多。** 专家缓存启动时为空（显卡刚好不够大）或
  另一个进程占用了显存。启动时会打印它持有多少专家；上次运行的日志位于
  程序旁边的 `strata-<model>.log`。
- **启动失败。** [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) 有对照表；
  `strata-<model>.log` 的前几行通常会指明原因。

## 文档

- **[docs/INDEX.md](docs/INDEX.md)** - 哪个页面回答什么问题，以及哪些数字是在此笔记本上实测的而非继承的。
- **[docs/BENCH-4GB.md](docs/BENCH-4GB.md)** - 此硬件的实测数据，附日志。
- **[docs/MEMORY-4GB.md](docs/MEMORY-4GB.md)** - 字节驻留位置：每个模型和每个 token 的显存、内存和 SSD 预算。
- **[docs/QUALITY-4GB.md](docs/QUALITY-4GB.md)** - 每次速度变更后运行的二十二提示词冒烟测试（必须运行的代码、工具循环、跨 18K token 的三个事实），及其发现。
- **[docs/MODELS-4GB.md](docs/MODELS-4GB.md)** - 十二个模型、大小、评分、来源。
- **[docs/INSTALL.md](docs/INSTALL.md)** - 所有安装问题、离线安装、其他驱动器。
- **[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** - 启动失败时该怎么办。
- **[docs/AI_SETUP.md](docs/AI_SETUP.md)** - 将安装交给 AI 助手。

## 状态

引擎、安装程序和服务器在参考笔记本上日常使用。诚实地说，这意味着：

- **在 4 GB 上实测：** Clover APEX-I-NANOPLUS 和 Flint UD-Q3_K_XL，32K 上下文。不是 131K，不是其他十个模型。
- **131K 上下文**是引擎及其构建支持的，也是发行版命名所依据的；此处仅测量了 32K。
- **新的速度设置**在同一会话中与之前的设置进行对比测量，无法复现的设置会被丢弃，而不是作为胜利记录下来。

## 致谢与许可

基于 VecSzn 的 [PolyStrata](https://github.com/VecSzn/PolyStrata) 构建，而后者又基于
Niko1221 的 [Strata](https://github.com/Niko1221/Strata) 构建。Qwen 引擎、安装程序、服务器和 Web 应用
来自 Strata；4 GB 引擎、其测量数据和本文档属于本分支。模型保留各自的
许可证——[docs/models/](docs/models/) 中的每个页面都注明了适用的许可证。

MIT，参见 [LICENSE](LICENSE)。
