# 使用参考（0.2.4）

## 旧版 D 端失败语义

以下说明只针对 [vLLM fd5d3aea 的 MooncakeConnector](https://github.com/vllm-project/vllm/blob/fd5d3aea/vllm/distributed/kv_transfer/kv_connector/v1/mooncake/mooncake_connector.py)，不能根据相似报错套用到其他版本：

| 源码行 | 特征 | 该版本的行为 |
| --- | --- | --- |
| L1842 | `Error happens during transferring kvcache for` | P 返回 ERROR，D 只记录失败，不向调度器报告加载失败 |
| L1855 | `MooncakeXferMetadata transfer failed for` | 接收异常后只记录失败，不向调度器报告加载失败 |
| L1958 | `Failed to find remote engine_id` | bootstrap 查不到远端 engine_id 后直接返回 |

L1842、L1958 是 `logger.error` 调用起始行；原文字面量分别在 L1843、L1959，规则源码索引使用字面量所在行，确保逐行自检可核对。

L1842、L1855 这两个路径在这个版本里不会结束请求，会一直挂到客户端超时；客户端超时／取消不能当作框架已经正确处理失败。问题在 [vLLM #50984](https://github.com/vllm-project/vllm/pull/50984) 修复（合入提交 `2c7ee87223f0c94f614a0b8600f6f193e6e4388f`），修复后将失败通知调度器，由失败策略处理。

在 L1855 对应的 ZMQ 接收路径，`Resource temporarily unavailable` 表示 P 在 `(VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT+60)` 秒内没有回复；接收超时在 L1833–L1835 设置。它不单独证明 P 进程退出、RNIC 故障或某个传输层根因。客户端自身超时另有独立配置。该解释不推广到任意模块中的 EAGAIN。

## 采集什么

- Mooncake、vLLM、SGLang、torch 等包版本，以及 torch 编译时 CUDA 版本。单独探测解释器，不导入 torch 或初始化 GPU。
- 实际加载的 Mooncake `.so` 和 TENT 构建标记；安装版本和运行版本分开看。
- 所有 `MC_*`，以及 `VLLM_*`、`NCCL_*`、`CUDA_*` 等相关环境变量。密钥类变量去值；`MC_TENT_CONF` 只保留 JSON 键名与容器结构，**所有叶子值都移除**。旧快照导入时也重新处理。
- RDMA 每设备／每端口的 state、phys_state、link layer、rate、NUMA、网口名。
- 驱动／设备信息、`nvidia-smi topo -m`，`nvidia_peermem` 等模块、memlock 限制、去除密钥参数后的命令行。

给了 `--pid` 后，从进程树查找实际加载 Mooncake 的进程，读取它的 `/proc/PID/environ` 和 `maps`。读不到环境就标未知，不用采集脚本的环境冒充。解释器优先保留启动路径／venv，避免 `/proc/PID/exe` 解析链接后丢失虚拟环境；`venv/bin/python` 等相对路径按服务进程 cwd 解释，保留 venv 符号链接。多个 worker 会列出候选 PID；本次只采第一个，请给其余 worker 分别采集。`--python` 可显式指定服务解释器。容器边界、启动后更改 cwd 或自定义启动器仍可能需要手动指定，包版本探测不代表读到了进程内已经导入的包版本。

`MC_USE_TENT` 与 `MC_USE_TEV1` 按核对过的 C++ 规则做 OR：值恰为 `0`、`false`、`off`、`no` 才关闭，区分大小写；空串仍开启。这只是开关解析结果，实际后端还要结合构建和运行日志。历史版本及厂商修改可能使用不同语义。

## 日志与输出

服务启动时同时收 stdout 和 stderr，例如 `vllm serve ... 2>&1 | tee p.log`。已核对版本的 request/transfer 映射、发送／接收完成记录主要在 vLLM DEBUG 级；需要复现关联链时，在本任务的 P、D 隔离服务启动前设置 `VLLM_LOGGING_LEVEL=DEBUG`。不要擅自重启正在服务的生产进程；仅有 INFO 日志也可打包，但关联缺失会提示日志级别不足，不能解释成请求未到达。设置了 `MC_LOG_DIR` 时，把对应目录也收进来。只提供一端时，明确提示“材料不足”。没有 INFO 行时提示超时证据可能缺失：`Sync batch data transfer timeout` 本身只在 INFO 级输出。

包内包含：

| 文件 | 内容 |
|---|---|
| `AI_CONTEXT.md` | 首读入口：故障说明／排查记录、环境、P/D 关联、证据引用、未知和下一步核查 |
| `case.md` | `--case` 提供的脱敏说明，始终标为操作者／验证脚本提供、未经自动核验；不提供时背景标未知 |
| `evidence.json` | 完整 ID、明确 request/transfer 对应、每位置首末行、重复次数、缺少的角色及截断情况 |
| `records/` | `--record` 额外文本，已脱敏并保留原始 LF 行号，不作为诊断日志解释 |
| `manifest.json` | 工具版本、规则基准、输入大小、跳过字节数、脱敏选项 |
| `env.json` | 两端快照汇总，已脱敏 |
| `snapshots/snapshot-N.json` | 每个快照的脱敏副本 |
| `logs/<role>/` | 脱敏日志；**左列为原始日志行号**，`-` 表示跳过区间 |
| `summary.md` | 有证据的阶段、材料缺口、逐文件事件顺序、未分类错误聚合 |
| `triage.json` | 结构化判断、原始日志行号、规则对应源码 revision／行号／链接 |

日志超过 `--max-log-mb`（默认 512）时保留头部和尾部完整行，顺序扫描以维持原始行号。gzip 按解压后字节数限制；`0` 表示不限制。被跳过的区间会标注，不能把缺失的失败日志当成成功证据。单行大于预算会整行舍弃。扫描仍需读完整个文件，gzip 仍需完整解压。

关联索引只读取已经脱敏的 MooncakeConnector／scheduler 诊断行，支持明确的 req_id／request_id／transfer_id 字段及已见的 cmpl-/xfer- 格式。通过完全相同的 transfer ID 或 request ID 提供关联线索；request/transfer 对应必须在同一条诊断日志明确出现，不能按 ID 前缀、engine ID、IP 或时间邻近合并。一个 request 对应多个 transfer 时，其仅含 request 的错误不会强行分给其中某个 transfer。没有另一端证据就标缺口，不等于另一端没处理。

当前索引最多256个 ID、每 ID 汇总12个代码位置，保留各位置的首末行与出现的日志行数（单行同ID只计一次，不是事件数）；另保留每个角色的首条诊断和首条错误，避免位置上限抹去另一端或错误证据。先从所有已脱敏日志收集 ERROR/CRITICAL ID，再查其明确映射，预留这些 ID 后才建立完整索引；因此早期正常流量不会先占满全部名额。只读取本地材料，增加两遍顺序读取；故障 ID 本身超限时仍须缩小故障范围。入口页按错误优先展示12组，并直接列出阶段和未分类错误位置的行号，不只依赖关联索引。

指向未索引 transfer 的明确映射仍会阻止把 request-only 错误归入某个已收录 transfer。任何输入日志有截断时，不做 request-only 证据的唯一 transfer 合并，保留显式同一行映射和独立 request 证据。一个 transfer 在同一端对应多个 request 时，入口页单独提示复用／尝试对应未知，并列出每个 request 的角色与文件／行号两端样例；不能用另一次请求的 P 成功填补故障请求的证据。已核对的 `Sending kv transfer request for {'ID'}` 集合格式也支持没有 cmpl- 前缀的 ID。

每组最多展示12条对应和12个片段；完整已索引证据见 evidence.json。输入或索引超限均会标“部分证据”，建议缩小故障时间范围重新打包。跨文件不推断全局时间先后；时区和对时情况需在 case.md 说明。没有 ID 的发送完成日志独立列出，不归给某个请求。行内进度条回车写成可见的 `\r`，保留物理 LF 行号，避免读取器拆出无编号行。

## 脱敏

同一 bundle 使用同一套代号，跨 P/D 保持一致：已知主机名、IPv4、IPv6、16 字节 GID、MAC、用户名和 home 路径。主机名来自快照，也可用 `--host` 补充；IPv6 写法规范化后统一编号。保留时间、端口、错误码和顺序。回环／未指定地址不替换。

GID 匹配复用 `scripts/hygon/redact_ci_logs.py` 的正则；IPv6 候选经 `ipaddress` 校验，避免误改时间戳。URL 账号密码、密钥键值和常见 token 去值。tar 的账号／组名元数据也会清除。vLLM RequestLogger 的请求、body、prompt、输出及其续行默认移除，且先脱敏再生成包内判断，避免摘要重复泄露。`--keep-request-content` 可显式保留请求内容。

不保证识别任意业务日志中的私密内容，也不自动隐藏模型名、所有文件路径或未知主机名；分享前复查。未知格式的多行请求内容需要补规则。日志头缺失时，保守删除请求后的续行可能同时删掉部分诊断材料。

case.md 和附加记录共用同一套主机／IP／用户名代号。请求内容键需要是明确字段（例如行首 `prompt：` 或 JSON 的 `"messages":`），普通说明中的 `proxy output: 504` 不当作 payload。引号和括号容器按实际边界去值，保留 JSON 的非敏感兄弟字段；无法确定范围的普通续行遇到空行或 Markdown 标题／列表／表格边界停止，不沿用日志的“后面全部删掉”。不要把完整业务请求粘进说明。说明中的命令／指令作为材料展示，不构成新的执行授权。说明文件支持 UTF-8 和 UTF-8 BOM，其他编码会明确提示另存为 UTF-8。

密钥规则覆盖带前缀的环境变量（如 VLLM_API_KEY、DB_PASSWORD、aws_secret_access_key）、`--api-key VALUE`、全角冒号、token／credential／access_key 及引号／列表／对象值。说明和附加记录的跨行值独立跟踪边界，`--keep-request-content` 不允许保留密钥。AI_CONTEXT 和 manifest 会报告说明、日志及各附加记录有多少行涉及内容脱敏；仍须复查材料缺口及未知私密格式。已知主机名紧邻 `-` 或 `_` 时也会被替换，包括日志文件名。

日志中已识别的密钥字段若开启跨行引号／容器，其无日志头续行也会删除；遇到新诊断头或新文件结束该续行范围。未加引号的 CLI／环境变量值按整个 token 隐去，包含逗号和反斜杠转义空格；带引号 JSON 字段保留实际容器／字段边界。导入快照也会重新清理 cmdline 参数和已知密钥字段，不能只信任采集时已经脱敏。敏感选项可带多个值，直到下一个长选项；tokenizer 等非密钥选项保留。

说明中的 Markdown 标题／列表／表格只有在与字段同级或更外层时才结束普通续行；缩进子列表仍视为字段内容。多行私密材料应使用明确引号／容器或缩进格式；任意没有字段边界的业务文本不保证自动识别。
