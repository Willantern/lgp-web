# Modern LGP 0.4

可扩展的线性遗传编程框架。Python 负责进化搜索；任务可以使用 Python、C++（CMake + SWIG）或 Java（JPype）执行程序并计算适应度。MATLAB 已移除。

## 0.4 的核心变化

- **Task 管完整评分**：Evaluator(task) 不要求解释器，任务可以直接连接自己的原生仿真环境。
- **原生循环完整运行**：内置回归和质点控制任务在 C++/Java 内部多次执行个体，一次返回适应度；没有每步回调 Python。
- **PreparedProgram**：每个个体准备一次，解析原语名称、检查索引和参数数目、分配参数缓冲；后续仿真复用。它是解释执行的数据，不是为每个个体编译动态库。
- **ProgramSpec**：完整标量程序描述包含原语名称、指令、输入/输出索引、只读寄存器和初值，可 JSON 往返。
- **确定性缓存**：新 Task 默认不缓存；声明完整且稳定的 cache_key 才缓存。计分调用和样本/仿真步数分别统计。
- 原生原语可以省略 Python func，不需要虚假的占位实现。

## 安装

基础安装只有标准库依赖，不会自动安装 C++ 编译器或 JDK：

```bash
python -m pip install -e .
python -m lgp regress --population 30 --generations 10 --output runs/python
```

完整开发环境：

```bash
python -m pip install -e '.[dev]'
python -m lgp doctor
```

可按需安装 .[cpp]、.[java]、.[control]、.[docs]、.[test]。C++ 需要 CMake ≥3.20、SWIG 4、C++17 编译器和当前 Python 开发头文件；Java 需要完整 JDK 17+ 和 JPype。Linux 示例：

```bash
sudo apt-get install g++ python3-dev openjdk-17-jdk
```

## 构建一次，执行完整原生仿真

```bash
python -m lgp build-runtime cpp --build-dir build/cpp --tests
ctest --test-dir build/cpp -C Release --output-on-failure
python -m lgp build-runtime java --build-dir build/java --tests

python examples/native_simulation.py --backend python
python examples/native_simulation.py --backend cpp --build-dir build/cpp
python examples/native_simulation.py --backend java --build-dir build/java
python examples/native_simulation.py --backend cpp --build-dir build/cpp --evolve
```

该示例是明确方程的教学仿真：位置和速度作为输入，加速度作为输出，每个回合运行 100 步，最小化位置误差与动作代价。它展示如何连接原生任务，不代表真实机器人基准。

Python 使用方式：

```python
from lgp import CppInterpreter
from lgp.fitness.evaluator import Evaluator
from lgp.tasks.native_tasks import PointMassTask

# primitives/registers 由任务配置；完整定义见 examples/native_simulation.py
with CppInterpreter(primitives, registers,
                    build_dir="build/cpp", auto_build=False) as runtime:
    evaluator = Evaluator(PointMassTask(runtime))
    evaluated = evaluator.evaluate(individual)
    print(evaluated.fitness, evaluator.metrics)
```

auto_build=False 只加载已有产物；不会调用 CMake/javac。默认 auto_build=True 保留教学便利，每个 runtime 首次 prepare() 构建并加载一次；提供 build_dir 可增量构建。修改源码后显式重建，再创建 runtime。不要并发写同一个构建目录。

回归 CLI 也已接入完整 Task：

```bash
python -m lgp regress --backend cpp --build-dir build/cpp --output runs/cpp
python -m lgp regress --backend java --build-dir build/java --output runs/java
```

## 自己的 C++/Java 环境如何接入

1. 在原生环境定义任务类：保存固定环境参数，score(PreparedProgram) 内部完成所有回合和评分。
2. C++ 在 include/lgp、src 增加文件，在 CMake 和 SWIG 接口中暴露类；Java 添加普通类并构建。
3. Python 写一个小 Task：evaluate(individual) 中取得 runtime.native_program(individual)，调用原生任务，返回 TaskResult。
4. 将 Task 交给原有 Evaluator 和 evolve；进化器无需修改。

不使用这些解释器的外部系统也能直接实现 Task。详见 [完整任务接入](docs/tasks.md)，含自定义 Task、缓存约定、错误处理和跨语言扩展实例。

## 目录与学习

```text
src/lgp/
  program/       原语、寄存器、个体、ProgramSpec、序列化
  runtime/       Python 执行器、构建和 SWIG/JPype 适配器
    _native/cpp/ include/lgp/、src/、bindings/、CMakeLists.txt
    _native/java/src/org/modernlgp/runtime/
  tasks/         TaskResult、兼容适配器、回归、控制和原生任务
  fitness/       缓存、计数、数值失败惩罚
  evolution/     初始化、选择、变异、交叉、进化
tests/           与 src 平级；含 tests/native/cpp 和 tests/native/java
examples/        基础执行、原语、回归、控制、完整原生仿真
docs/            MkDocs Material 文档
.github/workflows/  CI 和 GitHub Pages
```

每个 Python 源文件/示例、每个原生实现文件均有对应学习测试。原生源码只有一份，随 wheel 一起安装；测试随 ZIP 和源码分发包交付，不进入 wheel。

```bash
python -m pytest -q
python -m ruff check src tests examples
python -m mkdocs build --strict
python -m mkdocs serve
python -m build
```

GitHub 仓库 Settings → Pages → Source 选择 GitHub Actions，Documentation 工作流即可部署。此交付不包含线上发布。见 [部署说明](docs/deployment.md)。

## 兼容性与边界

0.3 的 Evaluator(runtime, objective) 和 run/run_many 仍可用；新代码优先 Evaluator(task)。旧构造默认缓存是调用者对固定确定性上下文的承诺，不能直接用于随机或可变环境。ControlTask 旧回调用 RuntimeTask(runtime, objective) 包装，默认关闭缓存。旧 schema 1/2 的 Individual JSON 仍可读取；它和包含寄存器布局的 ProgramSpec JSON 是两种用途的格式，不能互换。

适应度统一最小化。TaskResult.metrics 记录实际工作的可加总计数，缓存命中不重复计入。原生任务限纯标量；图像/对象原语可使用 Python 后端。每个 worker 使用独立 runtime/任务实例，PreparedProgram 内部工作区不能并发使用。close() 不关闭共享 JVM，也不能保证卸载所有已加载动态库。编译 timeout 不限制仿真运行时；需要硬超时或崩溃隔离时应由调用方使用进程边界。

本项目没有证明原生一定更快：仍有桥接、函数分派和输出数组分配成本。0.4 优先消除仿真每步桥接、重复名称解析和参数缓冲分配，尚未提供 JIT、向量化、分布式执行或复杂机器人基准。

参考 [Java LGP](https://github.com/Zhixing1020/Linear-Genetic-Programming-LGP-and-Applications) 与 [LGP4PY](https://github.com/Zhixing1020/LGP4PY_public) 的分层、有效指令分析和进化算子思想，独立实现。取舍见 [设计审查](docs/design-review.md)。MIT 许可见 LICENSE；实测见 [验证记录](docs/validation.md)。
