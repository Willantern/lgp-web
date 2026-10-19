# 学习路线与逐文件测试

推荐顺序：原语、指令、寄存器、个体、Python 执行、评价、初始化、选择、变异、交叉和进化。再阅读 runtime 的公共接口与原生桥接。

## Python 源码与测试

| 源文件 | 对应测试 |
| --- | --- |
| `src/lgp/__init__.py` | `tests/test_init.py` |
| `src/lgp/__main__.py` | `tests/test_main.py` |
| `src/lgp/cli.py` | `tests/test_cli.py` |
| `src/lgp/config.py` | `tests/test_config.py` |
| `src/lgp/evolution/__init__.py` | `tests/evolution/test_init.py` |
| `src/lgp/evolution/algorithm.py` | `tests/evolution/test_algorithm.py` |
| `src/lgp/evolution/crossover.py` | `tests/evolution/test_crossover.py` |
| `src/lgp/evolution/initialization.py` | `tests/evolution/test_initialization.py` |
| `src/lgp/evolution/mutation.py` | `tests/evolution/test_mutation.py` |
| `src/lgp/evolution/population.py` | `tests/evolution/test_population.py` |
| `src/lgp/evolution/selection.py` | `tests/evolution/test_selection.py` |
| `src/lgp/fitness/__init__.py` | `tests/fitness/test_init.py` |
| `src/lgp/fitness/evaluator.py` | `tests/fitness/test_evaluator.py` |
| `src/lgp/program/__init__.py` | `tests/program/test_init.py` |
| `src/lgp/program/analysis.py` | `tests/program/test_analysis.py` |
| `src/lgp/program/builtins.py` | `tests/program/test_builtins.py` |
| `src/lgp/program/individual.py` | `tests/program/test_individual.py` |
| `src/lgp/program/instruction.py` | `tests/program/test_instruction.py` |
| `src/lgp/program/primitive.py` | `tests/program/test_primitive.py` |
| `src/lgp/program/register.py` | `tests/program/test_register.py` |
| `src/lgp/program/serialization.py` | `tests/program/test_serialization.py` |
| `src/lgp/program/spec.py` | `tests/program/test_spec.py` |
| `src/lgp/runtime/__init__.py` | `tests/runtime/test_init.py` |
| `src/lgp/runtime/base.py` | `tests/runtime/test_base.py` |
| `src/lgp/runtime/build.py` | `tests/runtime/test_build.py` |
| `src/lgp/runtime/cpp.py` | `tests/runtime/test_cpp.py` |
| `src/lgp/runtime/errors.py` | `tests/runtime/test_errors.py` |
| `src/lgp/runtime/interpreter.py` | `tests/runtime/test_interpreter.py` |
| `src/lgp/runtime/java.py` | `tests/runtime/test_java.py` |
| `src/lgp/runtime/native.py` | `tests/runtime/test_native.py` |
| `src/lgp/runtime/registry.py` | `tests/runtime/test_registry.py` |
| `src/lgp/tasks/__init__.py` | `tests/tasks/test_init.py` |
| `src/lgp/tasks/base.py` | `tests/tasks/test_base.py` |
| `src/lgp/tasks/native_tasks.py` | `tests/tasks/test_native_tasks.py` |
| `src/lgp/tasks/control.py` | `tests/tasks/test_control.py` |
| `src/lgp/tasks/symbolic_regression.py` | `tests/tasks/test_symbolic_regression.py` |
| `src/lgp/utils/__init__.py` | `tests/utils/test_init.py` |
| `src/lgp/utils/logger.py` | `tests/utils/test_logger.py` |
| `src/lgp/utils/random.py` | `tests/utils/test_random.py` |

## 示例

| 示例 | 对应测试 |
| --- | --- |
| `examples/backends.py` | `tests/examples/test_backends.py` |
| `examples/basic_execution.py` | `tests/examples/test_basic_execution.py` |
| `examples/control.py` | `tests/examples/test_control.py` |
| `examples/native_simulation.py` | `tests/examples/test_native_simulation.py` |
| `examples/custom_primitive.py` | `tests/examples/test_custom_primitive.py` |
| `examples/symbolic_regression.py` | `tests/examples/test_symbolic_regression.py` |

## 原生源码学习

先读 `_native/cpp/include/lgp/primitive.hpp` 与 `registry.hpp`，再读 `src/primitives.cpp`、`src/extensions.cpp`、`src/registry.cpp` 和 `src/interpreter.cpp`。最后阅读 `bindings/lgp_runtime.i` 与 `CMakeLists.txt`，理解 SWIG 和 CMake 的职责。

C++ 每个实现文件都有根目录 `tests/native/cpp/test_*.cpp`，头文件由相同主题的测试覆盖。绑定层由 `tests/runtime/test_cpp.py` 检验直接调用时的参数和数值错误。

Java 按 `ScalarPrimitive → PrimitiveRegistry → CorePrimitives / ExtensionPrimitives → ScalarInterpreter` 阅读，每个 Java 源文件在根目录 `tests/native/java/org/modernlgp/runtime` 有对应 `Test*.java`。它们不依赖 JUnit，可用 java 命令直接运行。

`tests/runtime/test_source_mapping.py` 检查原生实现与测试对应关系，`tests/test_test_mapping.py` 检查 Python 文件与学习测试对应关系。

0.4 继续阅读 C++ prepared.hpp/prepared.cpp 与 tasks.hpp/tasks.cpp，Java PreparedProgram、RegressionTask、PointMassTask；每个实现仍有 tests/native 下的独立测试。先对照 Python PointMassTask 的积分方程，再阅读原生 score 循环，最后看 Python score_task 如何只跨语言调用一次。

## 运行

```bash
python -m pytest tests/program/test_primitive.py -v
python -m pytest tests/runtime/test_registry.py -v
python -m pytest tests/runtime/test_build.py -v
python -m pytest tests/runtime/test_conformance.py -v
python -m pytest -q
```

后端工具链缺失时集成测试会显示 SKIPPED。扩展测试实际复制原生源码并改变注册的原语名称，再用同一 Python 适配器构建和执行；重新构建测试检查同进程的新实例能够调用新二进制，旧实例保持原执行语义。

原生函数、Python 函数和跨语言转换应分别理解。框架可运行不代表复杂控制任务的效果已经验证。评价次数、缓存命中和环境交互步数要分开记录。
