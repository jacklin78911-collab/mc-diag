# 验证范围

当前源码来自 2026-10-17 自审修复后的 mc_diag 0.2.3。运行代码和测试与已验收交付一致；仓库补充了面向同事的说明、虚构示例及 CI，不上传真实现场日志或私密对照表。

## 已完成

- Python 3.9.25、3.11.15、3.12：各 65 项，64 通过、1 项真实扩展检查跳过；这些运行配置了 Mooncake／vLLM 源码自检。
- 外部审核提供的 15 个复现检查在上述三个解释器上通过。
- 本次真实故障归档的离线回放：39 条索引引用及入口页引用核对，原始 LF 行号保留；已知主机／IP／用户名及请求内容 canary 未出现在分享包。
- 早前已校验三个 PyPI 官方 Classic wheel（0.3.13.post1、0.3.11、0.3.0）的 SHA256，完成独立环境 CPU／TCP 传输与真实握手故障。
- 2026-10-16：单节点两张 H20、选定 RoCE RNIC，Classic 官方 0.3.13.post1 的 1 MiB CUDA 缓冲区完整 SHA256 读回一致；真实 vLLM MooncakeConnector P/D 与代理完成 HTTP 200 请求、16 token 输出，外部 KV token 计数 296。环境为 vLLM 0.28.1rc1.dev94+gfd5d3aea9、torch 2.13.0+cu130、Qwen3-0.6B。
- 同一现场仅主动停止本任务 P 后复现 D 控制元数据失败，日志有原始行号证据，操作背景单独记录；不据此宣称根因识别或自动恢复。

本仓库 CI 只运行离线单元测试和虚构示例，**不复现上述硬件实验**。历史硬件及真实日志证据保留在本地交付归档，不放入 GitHub。

## 未覆盖

- 跨节点 GPU／RDMA、TENT GPU／完整 PD、官方 TENT wheel。
- SGLang、MooncakeStoreConnector 的专用日志。
- 任意业务私密格式或所有历史／厂商定制构建。

## 可选源码与扩展检查

普通环境执行 `python3 -m unittest test_mc_diag -v` 会因缺少源码／扩展跳过 4 项。源码检查须使用匹配 revision 的 checkout：

```bash
MOONCAKE_SRC=/path/to/Mooncake VLLM_SRC=/path/to/vllm \
python3 -m unittest test_mc_diag.SelfCheck -v
```

默认核对 Mooncake `1d97a1d`、vLLM `e8e1e961`。其他已核对 revision 用 `MOONCAKE_REVISION`／`VLLM_REVISION` 指定，具体范围见代码；不要用不同提交的源码冒充对应验证。

```bash
MC_DIAG_TENT_SO=/path/to/tent/engine.cpython-312-x86_64-linux-gnu.so \
MC_DIAG_CLASSIC_SO=/path/to/classic/engine.cpython-312-x86_64-linux-gnu.so \
python3 -m unittest test_mc_diag.RealBuildTest -v
```

真实扩展测试需要匹配当前解释器 ABI 的两个真实构建，不提供时明确跳过。

## 修改记录

0.2.3 修复日志多行密钥、未加引号凭据的部分保留、导入快照中的命令行／结构化密钥、缩进排查记录中的敏感内容边界。0.2.2 已修复故障证据优先展示、DEBUG 材料缺口、transfer 复用提示、说明内容误删和带分隔符主机名脱敏。每项均有针对已发现问题的回归测试。
