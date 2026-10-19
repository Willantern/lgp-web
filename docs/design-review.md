# 参考仓库设计审查

本节基于实际拉取的源码，记录设计依据。不是对所有分支与实验的完整审计。

| 仓库 | 检查的提交 |
| --- | --- |
| [Java LGP and Applications](https://github.com/Zhixing1020/Linear-Genetic-Programming-LGP-and-Applications) | `12d0c745ba17d2b708f805860819f6964225f653` |
| [LGP4PY](https://github.com/Zhixing1020/LGP4PY_public) | `de6e100a1f750f05b56a2558e561d106dfa96616` |

## 具体检查路径

Java 仓库：

- `src/zhixing/cpxInd/individual/LGPIndividual.java`，特别是 `updateStatus` 与寄存器依赖。
- `src/zhixing/cpxInd/individual/reproduce/LGPMicroMutationPipeline.java`。
- `src/zhixing/cpxInd/individual/reproduce/LGPMacroMutationPipeline.java`。
- `src/zhixing/cpxInd/individual/reproduce/LGP2PointCrossoverPipeline.java`。

Python 仓库：

- `src/lgp/individual/lgp_individual.py`，特别是 `execute`、`preExecution`、`updateStatus`、`removeIneffectiveInstr`。
- `src/lgp/individual/reproduce/micro_mutation.py` 和 `macro_mutation.py`。
- `src/lgp/individual/reproduce/lgp_2point_crossover.py`。
- `src/lgp/individual/primitive/div.py`，以及 `src/ec` 与 `tasks` 的分离。

## 理性取舍

| 观察到的设计 | 优点 | 本项目决定及理由 |
| --- | --- | --- |
| 算法层与具体任务分开 | 便于把 LGP 用于回归、调度等不同任务 | 保留。0.4 通过完整 Task.evaluate 接入，支持原生环境内部评分 |
| 反向寄存器依赖与有效指令状态 | 识别不影响输出的指令 | 实现独立的纯程序分析，不把可变状态塞进 Individual |
| micro / macro mutation 分离 | 可以分别修改操作、读写寄存器与程序长度 | 提供显式模式，统一进行类型、长度和初始化校验 |
| 两点片段交叉并控制长度 | 适合线性程序且能避免无限增长 | 实现片段交换、长度边界和合法性验证 |
| ECJ 风格 Individual / Tree / Node 体系 | 支持已有框架的高级扩展和算子复用 | 当前轻量框架不继承整套对象图，用 opcode 与寄存器索引表达 |
| 个体同时保存执行寄存器与程序状态 | 在既有运行框架中方便复用 | 本项目把执行状态留给 runtime，减少并行时相互覆盖 |
| Python 仓库使用通配符导入与 `src.*` 路径 | 方便原仓库内部组织 | 改为安装包 `lgp.*` 与明确导入，支持 `pip install -e .` |
| 原语内部保护和裁剪 | 可抑制部分数值问题 | 明确每个原语语义，不在解释器里偷偷替换所有数学运算 |

这些决定是针对当前教学、可扩展和多后端目标的工程判断，不意味着参考架构在所有研究场景都更差。树形指令、控制流和复杂语义算子各有适用范围。

## 数值语义并未冒充一致

检查到 Python 参考仓库的 `div.py` 采用分母加 `1e-4` 的平滑保护并裁剪到约 `±1e6`。本项目 `div` 则在 `abs(b)>1e-12` 时计算 `a/b`，否则返回 `a`，不做通用裁剪。因此不能把本项目直接当成论文的同参数复现。

比较算法性能时，应先固定原语集合、保护函数、初始寄存器、评价预算、数据划分和随机种子。

## 上传代码的修正

保留 `Instruction`、`Individual` 和注册式 `PrimitiveSet` 的接口方向。移除 `primitive.py` 中会重复注册 `add` 的演示段，放到独立示例；补上零参数原语、未知类型验证器、有限数值约束、常量寄存器和非标量初始化检查。原来的 Interpreter 共享一个可变 RegisterFile，新 Python 执行器每次使用独立状态。

这是独立实现，没有打包上游源码，也没有继承上游所有高级算法。
