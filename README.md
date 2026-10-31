Antigravity 总是“无限重新思考”并爆 Model Unavailable？扒开 transcript.jsonl 底层日志，我发现了云端网关的“300 秒铡刀”

如果你在 Google Antigravity (AGY) 中开启了高阶推理模型（High Thinking），并让它帮你撰写一份万字深度报告（Artifact）或重构一个大型代码文件，你很大概率遇到过下面这个令人血压飙升的场景：

你在 Antigravity 对话框下达了一个复杂的长文档/多图表生成任务；
Agent 开始进入漫长的 Thinking...（思考了足足三四分钟）；
眼看它马上就要调用 write_file 把 Artifact 写完落盘了，界面突然闪出一行报错：Error: The stream was interrupted. Please continue the task you were working on. 或者弹窗提示 model unavailable；
更崩溃的是，Antigravity 的自动恢复机制触发了重试，Agent 又从头开始重新思考……如此循环往复了近 20 分钟，最终 Artifact 目录里一个字都没保存下来！

很多 Antigravity 用户的第一反应是：“是不是我的本地网络代理（Clash / V2Ray / Tun 模式）断流了？把代理的 Timeout 改大一点是不是就好了？”

今天这篇文章，我们就直接扒开 Antigravity 本地隐藏的底层会话运行日志（transcript.jsonl），用精确到秒的数据告诉你：真相到底是什么，为什么改本地代理毫无作用，以及如何用一条 Antigravity 全局规则（Global Rule）在 30 秒内永久根治这个问题。

一、 案发现场还原：Antigravity 底层日志里的“幽灵秒表”

很多人不知道，Antigravity 会把每一次对话的完整推理轨迹、Token 消耗、缓存命中以及底层报错，实时记录在本地的 brain 目录中：

Windows 路径：C:\Users\<用户名>\.gemini\antigravity\brain\<conversation-id>\.system_generated\logs\transcript.jsonl
macOS / Linux 路径：~/.gemini/antigravity/brain/<conversation-id>/.system_generated/logs/transcript.jsonl

为了搞清楚那 20 分钟里到底发生了什么，我直接打开了当前会话的 transcript.jsonl。结果，几组精确得令人发指的时间戳直接暴露了真凶（以下为真实日志摘录）：

json
// 第 1 次尝试：耗时 3分04秒（184秒）后被掐断
{"step_index":15, "type":"PLANNER_RESPONSE", "created_at":"06:50:18Z", "input_tokens":28335, "cache_read_tokens":0, "output_tokens":3138}
{"step_index":16, "type":"ERROR_MESSAGE",    "created_at":"06:53:22Z", "error":"The stream was interrupted. Please continue..."}
// 第 2 次尝试：自动重试，耗时 5分37秒（337秒）后再次被掐断
{"step_index":17, "type":"PLANNER_RESPONSE", "created_at":"06:53:26Z", "input_tokens":28351, "cache_read_tokens":0, "output_tokens":1548}
{"step_index":18, "type":"ERROR_MESSAGE",    "created_at":"06:59:03Z", "error":"The stream was interrupted. Please continue..."}
// 第 3 次尝试：再次自动重试，耗时 5分09秒（309秒）后第三次被掐断
{"step_index":19, "type":"PLANNER_RESPONSE", "created_at":"06:59:10Z", "input_tokens":28367, "cache_read_tokens":0, "output_tokens":108}
{"step_index":20, "type":"ERROR_MESSAGE",    "created_at":"07:04:19Z", "error":"The stream was interrupted. Please continue..."}

看到规律了吗？

第 1 次中断：184 秒（刚好卡在 180 秒 / 3 分钟 阈值）
第 2 次中断：337 秒（卡在 300 秒 / 5 分钟 阈值 + 重连缓冲）
第 3 次中断：309 秒（再次精准卡在 300 秒 / 5 分钟 阈值）

这不是随机的网络抖动，而是一把精准无比的“定时铡刀”——只要单次请求耗时触碰到 180 秒或 300 秒，连接就会被准时斩断！

二、 硬核拆解：什么网关在挥刀？为什么流式传输也会超时？

在排查这个问题时，绝大多数开发者会有三个核心疑问。我们逐一拆解：

疑问 1：为什么改本地代理软件的 Timeout 参数没用？

因为网络链路遵循经典的“木桶效应（最短板原理）”。从你的客户端到云端算力卡，中间串联了三道关卡：

如果你的连接在 30 秒或 60 秒断开，那是你本地代理的问题；但当连接已经撑到了 180 秒～300 秒 才断，说明你本地代理早就放行了。真正挥刀的是云端 API 网关（如 Google Front End / Envoy）和后端的 TPU 推理调度器。

云端架构师为了防止慢请求或死循环无限期霸占昂贵的算力卡（防止并发连接池耗尽引发机房雪崩），在服务端网关配置里硬编码（Hardcoded）了 180s～300s 的单次流最大存活寿命（Max Stream Duration）。服务端写死的规则，你在客户端怎么改代理参数都无济于事。

疑问 2：流式输出（Streaming）不是一直在吐字吗？为什么还会超时？

这是最大的认知误区。网关超时分为两种：

空闲超时（Idle Timeout）：一段时间没数据传输才断开；
总寿命硬上限（Hard Deadline）：不管管道里有没有数据在流动，单次 RPC 请求从发起的那一秒开始倒计时，满 300 秒格杀勿论。

当你在高阶推理模型（High）下要求生成一份万字报告时，模型在单次回合（Single Turn）里干了三件极其耗时的事：

Prefill（重读上下文）：阅读此前累积的近 3 万 Token 历史记录；
Thinking（长思考链）：在脑海里先推演了 3,100+ Token 的英文大纲；
Tool Call（拼装巨型文件）：开始逐字生成 write_file 工具里的 Content 参数（上万字的 Markdown + 多张复杂 Mermaid 图表）。

三项叠加，总生成量超过 10,000+ Output Tokens。按高阶模型每秒 30～35 Token 的生成速度，跑完需要 320 秒以上——还没等模型写完，第 300 秒的云端铡刀就落下了。

疑问 3：为什么断流后会陷入“无限重新思考”的死循环？

这涉及到两个致命机制的叠加：

工具调用（JSON）的“全有或全无（All-or-Nothing）”特性：普通聊天吐出一半被掐断，你还能看到前一半文字；但 write_file(Path="...", Content="...") 是一个严格的 JSON 函数调用。哪怕模型辛苦写了 4 分 59 秒、完成了 99% 的内容，只要最后那个闭合引号和花括号 "} 还没吐出来，JSON 就是非法的，工具根本无法执行，硬盘上一个字都不会落盘！
上下文缓存（Prompt Cache）雪崩归零：仔细看上面日志里的 "cache_read_tokens": 0！因为请求是异常中断的，该轮的上下文缓存未能正常刷写。系统自动触发重试时，模型不仅要从第 0 个字重新写报告，还要从头重新计算那 2.8 万个历史输入 Token，导致重试一次比一次慢，彻底锁死在“5 分钟超时死循环”里。
三、 破局之道：用“分片传输（Chunking）”思想改造 Antigravity 写入策略

既然云端网关那堵 300 秒的硬墙 是服务端写死的，我们在 Antigravity 客户端侧唯一的工程正解是什么？

答案就藏在计算机网络的基础教材里——就像网页上传一个 10GB 的高清视频，当服务端限制单个 HTTP 请求不能超过 5 分钟时，唯一的解法就是“分片上传（Chunked Upload）”。

映射到 Antigravity 写大型 Artifact 报告或长代码文件的场景中，就是利用 Antigravity 内置 write_file 工具原生支持的 Append: true（追加模式），将“单次巨型 write_file”重构为“多次轻量级分片追加写入”：

真实实测对比数据

在我将生成策略改为“3 批次分片追加”后，同样一份包含 8 张 Mermaid 图表、3 个跨国历史样本的万字深度报告，底层日志表现发生了质的飞跃：

指标维度	单次巨型写入（改造前）	分片追加写入（改造后）
单次请求耗时	184s ～ 337s（连续触发网关斩断）	15s ～ 25s / 次（远低于 300s 阈值）
文件落盘耗时	失败（重试 4 次耗时 19 分钟无果）	每次仅需 2 秒，总计不到 1 分钟全部完成
Prompt 缓存命中（cache_read_tokens）	0（断流导致缓存失效，每次重算）	高达 29,527 Tokens（100% 命中缓存，零预热延迟）
成功率	0%（死循环）	100% 稳定闭合
四、 抄作业指南：在 Antigravity 中 30 秒配置全局防超时规则

难道以后每次在 Antigravity 里写长文，我们都要手动在提示词里叮嘱一句“请分段写”吗？当然不用！

我们可以直接利用 Antigravity 的 Customization System（自定义规则系统），把这套“分片防超时协议”设置为 always_on 全局规则。配置完成后，Antigravity 在所有工作区、所有新建对话中都会将其作为最高优先级的 <user_rules> 自动加载执行。

方法 1：写入 Antigravity 全局规则目录（最推荐）

直接在 Antigravity 的全局规则目录中创建一个规则文件（创建后你可以在 Antigravity 左侧边栏点击 Skills & Customizations 
→
→ Rules 看到它）：

Windows 路径：C:\Users\<你的用户名>\.gemini\config\rules\chunked-write.md
macOS / Linux 路径：~/.gemini/config/rules/chunked-write.md

NOTE

注意避免重复创建：Antigravity 同时支持读取 ~/.gemini/GEMINI.md（显示为 user_global）和 ~/.gemini/config/rules/*.md。由于系统是按文件路径去重的，千万不要在两个路径下同时放同一条规则，否则面板里会出现两条一模一样的规则！只创建下面这一个 chunked-write.md 文件即可。

将以下 Markdown 规则完整复制进去保存即可（或者直接让 Antigravity 帮你创建该文件）：

markdown
trigger
:
always_on
description
:
防止长文档或大文件单次流式生成超过云端网关 3-5 分钟超时阈值
大文件与长报告分段写入规则 (Chunked Write Anti-Timeout)
禁止单次巨型写入：当需要创建或输出超过 2,500 字的长篇报告、包含多个 Mermaid 图表的文档（Artifact）或超过 300 行的代码文件时，严禁在单次 write_file 工具调用中一次性生成全部内容。
强制分片追加（Chunked Append）：
必须将长文档按逻辑章节拆分为 2～4 个独立批次；
第 1 批次使用 write_file（Overwrite: true）写入第一部分（控制在 2,000 字以内）；
后续批次依次调用 write_file（Append: true）分段追加剩余章节，确保单次工具调用在 30～60 秒内完成闭合。
精简分段思考链：在执行分段写入的每个回合中，保持内部思考（Thinking）简短直接，不要在单次工具调用前进行数千 Token 的重复推演。
方法 2：使用内置 /learn 命令一键注入

如果你不想手动找文件夹，可以直接在聊天框输入：

/learn 以后在生成超过2500字的长篇报告、多图表文档或超过300行的代码文件时，严禁单次 write_file 一次性输出全文，必须拆分为2-4个批次，第1批用 Overwrite: true 创建，后续批次用 Append: true 分段追加，确保单次请求在60秒内完成。

方法 3：日常重度使用的两个配套习惯
长对话适时运行 /compact：当单次对话已经累积了数万 Token，在开启下一个大型生成任务前，输入 /compact 压缩历史上下文，能大幅缩短 Prefill（首包预热）时间。
善用 Subagents（子代理）并发：遇到超大型工程，让主 Agent 将不同模块分发给多个子代理并行撰写，每个子代理上下文极轻，彻底告别卡顿。
结语

很多时候，我们在使用最前沿的 AI 大模型时遇到的诡异 Bug，剥开外壳一看，底下依然是计算机科学最朴素的底层逻辑——网关超时控制、RPC 序列化边界、缓存失效雪崩与分片传输（Chunking）。

希望这篇从真实崩溃日志里扒出来的排查笔记，能帮你彻底告别 The stream was interrupted 和 model unavailable 的烦恼！
