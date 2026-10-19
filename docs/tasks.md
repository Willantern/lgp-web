# 完整任务与原生仿真

## 执行程序与评价任务是两件事

runtime.run 返回程序输出，例如一个加速度；Task.evaluate 返回整个任务的适应度，例如四个回合的平均控制代价。0.4 允许环境和程序执行都留在 C++/Java，Python 每个候选只发送一次评分请求。

| 接口 | 输入 | 返回 |
| --- | --- | --- |
| runtime.run | Individual、本次观测 | 程序输出元组 |
| PreparedProgram.execute | 按输入寄存器顺序排列的标量 | 输出数组 |
| 原生任务 score | PreparedProgram | 完整任务评分 |
| Task.evaluate | Individual | TaskResult(fitness, metrics) |
| Evaluator.evaluate | Individual | 附带 fitness 的新 Individual |

## 不依赖 runtime 的自定义 Task

```python
from lgp.tasks.base import TaskResult
from lgp.fitness.evaluator import Evaluator

class MyTask:
    # 只有确定性任务才能填键；随机环境保留 None。
    cache_key = None

    def __init__(self, simulator):
        self.simulator = simulator

    def evaluate(self, individual):
        # simulator 自己负责程序转换、环境运行和评分。
        fitness, steps = self.simulator.evaluate(individual)
        return TaskResult(fitness, {"steps": steps})

# evaluator = Evaluator(MyTask(simulator))
```

这也是外部进程、现有 C++ 仿真器或其他语言系统的接入点，不要求继承框架基类。Task 接收 Individual；当原生环境还需要布局和初值时，用 ProgramSpec.bind 或 runtime.native_program 形成完整描述。

## 内置原生回归任务

```python
from lgp import CppInterpreter, RegisterFile
from lgp.program.builtins import scalar_primitives
from lgp.tasks.symbolic_regression import polynomial_task
from lgp.tasks.native_tasks import RegressionTask
from lgp.fitness.evaluator import Evaluator

p = scalar_primitives()
r = RegisterFile(("scalar",) * 5, readonly=(0,))
with CppInterpreter(p, r, build_dir="build/cpp", auto_build=False) as runtime:
    evaluator = Evaluator(RegressionTask(runtime, polynomial_task()))
    # evaluator.evaluate(individual)
```

polynomial_task 是生成 x*x+x 样本的便捷函数；SymbolicRegression 存储固定输入、目标和输入寄存器编号。RegressionTask 将它变为统一 Task：Python 后端使用参考计算，C++/Java 后端把数据集传入原生任务对象一次，再在原生端计算全部误差。

## 完整控制仿真

完整可运行代码见 examples/native_simulation.py。每个回合从固定位置与零速度开始：

- 观测：位置 x、速度 v，依输入寄存器升序传入。
- 动作：程序唯一输出，裁剪到 [-1,1]，作为加速度 a。
- 更新顺序：v += dt*a，然后 x += dt*v。
- 代价：更新后 x*x + 0.01*a*a，对全部回合与步数取平均。
- 每次策略调用恢复寄存器初值；每回合重置环境状态。

默认四个初始位置 (-1,-0.5,0.5,1)、100 步、dt=0.05。Python 版本用于语义对照；C++/Java 版本在原生 score 内完成整个双重循环。测试验证三种实现的一致性、重复评价重置、不同个体之间的状态隔离，以及原生评分不调用 Python run/run_many。

## 为自己的原生环境增加 Task

C++ 可以照着以下两个文件增加任务类：

- src/lgp/runtime/_native/cpp/include/lgp/tasks.hpp
- src/lgp/runtime/_native/cpp/src/tasks.cpp

核心方法是 double MyTask::score(lgp::PreparedProgram& program)。初始化构造函数保存场景/数据，score 内重置环境，循环读取观测、调用 program.execute、施加动作、计算最终 fitness。不要在 score 每步调用 Python。

新增文件时加入 CMake 的 lgp_core 源文件列表，并在 bindings/lgp_runtime.i 的编译包含区和 %include 区暴露头文件；第三方仿真库也在 CMake 中链接。构建后 Python Task 可以这样调用：

```python
from lgp.tasks.base import TaskResult

class MyCppTask:
    cache_key = None

    def __init__(self, runtime, scenario_path):
        self.runtime = runtime
        self.native = runtime.native_class("MyTask")(str(scenario_path))

    def evaluate(self, individual):
        # score_task 提供异常映射和句柄准备，支持任意含 score(handle) 的任务对象。
        fitness = self.runtime.score_task(self.native, individual)
        return TaskResult(fitness)
```

这是接口示意，MyTask 需要由用户实现并暴露，不是内置类。若任务类位于命名空间，使用 SWIG 导出的 Python 类名。

Java 同样实现 public double score(PreparedProgram program)，构造函数保存固定场景，通过 runtime.native_class("your.package.MyTask") 使用同一个类加载器加载。可在 JavaInterpreter(..., classpath=("libs/simulator.jar",)) 中加入第三方 jar 或类目录；这些路径必须存在，框架不管理 Maven/Gradle 依赖。JPype 参数转换由用户 Task 负责；内置 create_task 只是两个教学任务的转换便捷函数，不是扩展任务注册的必要步骤。

## 完整程序描述与保存

```python
from lgp import ProgramSpec

# spec = ProgramSpec.bind(individual, primitives, registers, "cpp")
# text = spec.to_json()
# restored = ProgramSpec.from_json(text)
# handle = runtime.prepare_spec(restored)
# outputs = handle.execute([0.2, 0.0])
```

ProgramSpec 包含原生操作名称、目的/源索引、输出、全部初值、输入和只读索引、后端和 schema。它不包含函数实现、数据集或仿真配置；复现实验仍须固定编译源码与任务配置。backend 必须与 runtime 匹配。原生函数名/参数数目在准备时继续检查。Individual 的旧 JSON 仍用于保存进化基因型，两种 JSON 不能互换。

## 缓存、失败与资源

新 Task 未声明 cache_key 或设置为 None 时，Evaluator 默认不缓存。cache=True 也不能绕过这项要求。声明键意味着调用者保证完整上下文的确定性；键应包含数据版本、固定种子、场景配置和实现版本。内置任务使用不可变配置和 runtime 局部上下文；随机训练种子必须由自定义任务明确管理。

键在两次评价之间改变会清空评分缓存；评价过程中改变则报错。框架无法自动发现隐藏在外部仿真器里的可变状态。固定了原语，并不能证明任务确定性。

TaskResult 只允许有限适应度与有限数值 metrics。Evaluator 将算术异常和 NativeExecutionError 转为 failure_fitness；配置错误、TypeError、编译和依赖错误会抛出，避免把接口错误当作劣质个体。Python scalar 验证器的 TypeError 也明确抛出；自定义任务若把某种非有限仿真结果视为可惩罚失败，应主动抛 ArithmeticError。

evaluations 是实际评分调用数；metrics 用于可加总工作计数（samples、steps、episodes），缓存命中不会重复累加。不要把均值这类不可直接相加的统计量当作工作计数。原生程序准备与任务初始化不计入 steps，计时应另行测量。

每个 worker 独立实例。句柄中的寄存器/参数缓冲可变，禁止同时执行同一实例。不要在 runtime.close() 后继续使用导出的原生对象。timeout 只限制编译，原生无限循环或崩溃需要应用层进程隔离。

## 从 0.3 迁移

推荐 Evaluator(RegressionTask(runtime, data))。旧 Evaluator(runtime, objective) 保留兼容，默认缓存仍视为调用者对固定确定性环境的承诺。

现有 ControlTask 使用 RuntimeTask(runtime, control_task) 包装即可接入新接口，默认无缓存；仅当种子、环境配置和实现都固定时才传 context_key。它仍在 Python 循环中调用 Gym 环境，不会自动迁移成原生环境。完整原生环境应实现上文 score(handle) 接口。
