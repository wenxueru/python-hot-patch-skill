# Python Hot Patch Skill

[English](./README.md) | [简体中文](./README_zh.md)

一个用于指导 Codex **安全地热修补正在运行的 CPython 进程**的 skill。它面向无法轻易重启、需要保留昂贵内存状态，并且变更能够收敛为小范围 Python 对象或方法修改的场景。

该 skill 采用 `gdb`、临时共享库和 `Py_AddPendingCall`：调试器只负责将回调加入待处理队列，真正的 Python 修改在解释器主线程到达安全点后执行。

## 适用场景

- 重启会丢失代价高昂的内存状态。
- 修改目标明确，且可以表达为现有 Python 对象或方法的小范围变更。
- 可以在操作前定义并独立验证进程存活、语义状态、吞吐量或并发度等验收条件。
- 热修补完成后，能够同步修改磁盘上的源码或配置，避免下次重启回退。

以下情况应优先重启，或使用正式控制接口、信号处理器和已有调试端点：

- 修改解释器内部、对象布局或原生扩展状态。
- 修改模型结构、优化器状态或跨线程不变量。
- 无法唯一识别目标对象、确认旧状态或可靠验证结果。

## 核心安全约束

- 每次附加进程前必须获得明确授权。
- 不得从任意暂停线程直接调用 `PyRun_SimpleString`，否则可能因线程未持有 GIL 而死锁或破坏解释器状态。
- 补丁必须具备幂等保护，并记录包含旧值和新值的 marker。
- `Py_AddPendingCall` 返回 `0` 只表示成功排队，不代表补丁已经执行。
- 回调结果未知时只等待 marker 或进程退出，不得重复附加或再次入队。
- marker 之外还必须独立验证应用行为、错误日志和资源使用情况。

完整操作规范见 [`SKILL.md`](./SKILL.md)。

## 安装

将仓库复制或链接到 Codex skills 目录：

```bash
ln -s /path/to/python-hot-patch-skill ~/.codex/skills/python-hot-patch
```

重新启动或刷新 Codex 会话后，skill 可根据任务描述自动匹配，也可以显式使用 `$python-hot-patch`。

## 使用示例

```text
使用 $python-hot-patch 检查 PID 12345 的 asyncio 并发限制是否可以从 8 安全提高到 12。
先制定验收标准并检查环境；实际 attach 前必须再次向我确认。
```

skill 会优先评估更安全的替代方案。只有满足安全边界时，才会准备临时共享库和待执行补丁；实际 `gdb attach` 仍需要即时授权。

## `asyncio.Semaphore` 特别说明

提高并发上限时，应在事件循环线程中对原有 semaphore 调用恰好 `new - old` 次 `release()`，同时更新其所属的限制字段。不要替换 semaphore 对象，因为已有 waiters 仍绑定在原对象上。

## 仓库结构

```text
.
├── README.md
├── README_zh.md
├── SKILL.md
└── agents/
    └── openai.yaml
```

- `SKILL.md`：Codex 加载的核心工作流和安全边界。
- `agents/openai.yaml`：skill 的界面显示信息。

## 维护原则

- 保持 skill 聚焦于小范围、可验证的 CPython 在线修改。
- 新增规则应来源于真实风险或已观察到的失败，不堆积泛化限制。
- 修改后运行 Codex `skill-creator` 提供的 `quick_validate.py` 校验 `SKILL.md`。
- 不在仓库中保存进程转储、生产凭证、临时共享库或带有敏感状态的 marker。
