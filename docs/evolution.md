# 进化与评价

## 完整 API 示例

```python
from lgp import PythonInterpreter, RegisterFile
from lgp.program.builtins import scalar_primitives
from lgp.config import EvolutionConfig
from lgp.evolution.initialization import ProgramFactory
from lgp.evolution.algorithm import evolve
from lgp.fitness.evaluator import Evaluator
from lgp.tasks.symbolic_regression import polynomial_task
from lgp.tasks.native_tasks import RegressionTask

config = EvolutionConfig(population_size=100, generations=50, seed=42)
p = scalar_primitives()
r = RegisterFile(("scalar",) * 5, readonly=(0,))
factory = ProgramFactory(p, r, outputs=(4,),
                         min_length=config.min_length, max_length=config.max_length)
evaluator = Evaluator(RegressionTask(PythonInterpreter(p, r), polynomial_task()))
result = evolve(factory, evaluator, config)
print(result.best.fitness)
```

## 算法步骤

初始化合法指令序列，评价初始种群。每代保留精英，通过锦标赛选择父代，以给定概率交叉和变异，再评价子代。达到代数、目标适应度或评价次数限制时返回最优程序及完整日志。

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| population_size | 100 | 种群大小 |
| generations | 50 | 初始评价之后最多产生的代数 |
| min_length / max_length | 1 / 24 | 指令条数边界 |
| tournament_size | 3 | 每次锦标赛抽样数，允许重复抽样 |
| elitism | 1 | 原样保留的最好个体数量 |
| crossover_rate | 0.7 | 使用交叉的概率 |
| mutation_rate | 0.9 | 对子代执行变异的概率 |
| seed | 42 | 该次搜索的局部随机数种子 |
| target_fitness | 1e-12 | 达标停止；None 表示关闭 |
| max_evaluations | None | 实际目标函数调用上限，必须覆盖初始种群 |

适应度统一按小到大排序，完全相同时偏好短程序。排名选择作为独立 API 提供，默认进化循环使用锦标赛。

## 操作合法性

初始化只读取已初始化寄存器。微变异可以改变 opcode、源寄存器或目标寄存器；宏变异插入或删除指令。两点交叉交换父代的连续片段。生成子代后重新检查类型、长度、只读属性和初始化依赖。有限次尝试失败时返回没有适应度的父代副本，避免无限重试。

当前不保证每次变异都改变程序，也不保证新程序行为不同。输出寄存器由 factory 固定；不同布局或不同输出约定的父代不能直接混交。

## 缓存与预算

`Evaluator(task)` 不要求 runtime。新 Task 只有提供非 None 的确定性 cache_key 时才默认缓存；上下文变更会清空旧缓存。程序键包含指令和输出，不包含适应度。缓存命中不增加 evaluations，也不增加 metrics 中的实际工作计数。旧 Evaluator(runtime, objective) 仅作兼容。

一次未缓存目标函数调用计为一次评价，失败也计数。初始种群包含在预算内。达到上限时，用已有已评价个体补齐当前种群，不多调用一次目标函数。

回归的一次评价涉及全部样本；控制的一次评价可能涉及多个 episode 和许多环境 step。论文对比应分别报告个体评价次数和样本/环境交互次数。

## 数值失败

`failure_fitness=1e100` 是默认有限惩罚。Evaluator 捕获 ArithmeticError 和 NativeExecutionError；TypeError、缺少后端、编译错误、程序静态不合法和任务配置错误明确抛出。Task.evaluate 必须返回 TaskResult。

使用自定义目标时需要确保失败惩罚大于正常目标值。使用随机环境时应固定评价种子，或关闭缓存。`evolve()` 每次开始会重置 evaluator 的缓存与计数，避免上一次搜索污染独立重复实验。

## 保存与复现

`save_run()` 写出配置、程序和每代日志。`load_program()` 读取 JSON，校验原语顺序和签名，再由 runtime 验证实际布局。JSON 不保存函数实现、数据集或环境安装，请同时固定这些实验条件。
