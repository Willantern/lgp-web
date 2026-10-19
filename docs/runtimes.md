# runtime 维护与扩展

runtime 把候选程序转换为输出。0.4 提供 Python、C++ 和 Java，三者共享 run()、run_many()、close() 与上下文管理接口。完整任务适应度由 Task 负责，见 [完整任务与原生仿真](tasks.md)。

## 分层职责

| 文件或目录 | 职责 |
| --- | --- |
| base.py | RuntimeBase、输入状态与生命周期、RuntimeCapabilities |
| interpreter.py | 通用 Python 执行与 Runtime 协议 |
| native.py | 原生标量限制、程序编码缓存与输出检查 |
| build.py | 调用 CMake 或 javac 构建已存在的源码 |
| cpp.py | 加载 SWIG 扩展，调用 C++ 函数 |
| java.py | JPype 桥接与独立类加载器 |
| registry.py | 注册和创建后端 |
| errors.py | 依赖缺失、编译失败、运行失败和不支持的原语 |
| _native/cpp | 可独立构建和测试的 C++ 库 |
| _native/java | 可独立编译和测试的 Java 类 |

原生适配器继承共同的 RuntimeBase，不再继承 PythonInterpreter。增加一种后端不用修改进化、选择、变异和任务层。

## C++ 源码组织

C++ 核心函数写在 .cpp/.hpp 中，由 CMake 编译。SWIG 包装 interpreter.hpp、prepared.hpp、tasks.hpp；启用 builtin 类包装，Python 可直接构造原生程序和任务对象。新增任务不需要修改进化器。

| 源文件 | 修改场景 | 对应测试 |
| --- | --- | --- |
| include/lgp/primitive.hpp | 改变标量原语接口 | test_primitives.cpp |
| include/lgp/registry.hpp、src/registry.cpp | 改变原生名称查找 | test_registry.cpp |
| src/primitives.cpp | 修改公共数学原语 | test_primitives.cpp |
| src/extensions.cpp | 增加领域原语 | tests/native/cpp/test_extensions.cpp |
| include/lgp/interpreter.hpp、src/interpreter.cpp | 修改批量执行语义 | test_interpreter.cpp |
| bindings/lgp_runtime.i | 改变跨语言类型、异常映射 | tests/runtime/test_cpp.py |
| CMakeLists.txt | 加源文件、编译选项或第三方依赖 | CTest 与 Python 集成测试 |

源码路径位于 src/lgp/runtime/_native/cpp。C++ 测试统一位于根目录 tests/native/cpp；Java 测试位于 tests/native/java/org/modernlgp/runtime。根 CMakeLists.txt 只是方便从仓库根目录进入构建，核心来源只有一份。

## 完整 C++ 扩展例子

1. 在 src/extensions.cpp 中，给 register_extension_primitives() 增加：

```cpp
r.add("deadzone", {2, [](const auto& a) {
    return std::abs(a[0]) < std::abs(a[1]) ? 0.0 : a[0];
}});
```

文件头增加 #include <cmath>。参数数组中 a[0] 是信号，a[1] 是阈值；二表示操作的参数数量。

2. 在 Python 注册同一含义的操作：

```python
p.register("deadzone", ("scalar", "scalar"), "scalar",
           lambda x,t: 0.0 if abs(x)<abs(t) else x,
           implementations={"cpp":"deadzone"})
```

3. 为阈值内外和正负值编写原生测试，再构建：

```bash
python -m lgp build-runtime cpp --build-dir build/cpp --tests
ctest --test-dir build/cpp -C Release --output-on-failure
```

4. 用新实例加载构建结果：

```python
with CppInterpreter(p, registers, build_dir="build/cpp") as runtime:
    runtime.prepare()
    result = runtime.run(program, inputs)
```

prepare() 明确承担构建与加载；run() 也可以按需第一次调用它。构建后同一实例执行不同个体时复用一个 C++ 库，传入操作名称和寄存器索引。名称在 C++ 批量执行入口解析为函数，不在每条指令上调用 Python。

使用外部源码副本时传 source_dir="/path/to/copied/cpp"。把自定义文件加入 CMake 的 lgp_core 源文件列表即可；第三方库使用正常的 target_link_libraries() 管理。

C++ 加载时将二进制按内容摘要复制到独立路径，避免重新编译后在同一 Python 进程意外复用旧库。已经创建的实例继续执行已加载版本，修改源码后请创建新实例。不要并发写同一个 build_dir。

## Java 扩展

文件位于 _native/java/src/org/modernlgp/runtime：

| 类 | 作用 |
| --- | --- |
| ScalarPrimitive | 可调用原语接口 |
| PrimitiveRegistry | 名称、参数数目与函数注册 |
| CorePrimitives | 公共原语 |
| ExtensionPrimitives | 新领域原语 |
| ScalarInterpreter | 批量执行、边界与有限数值检查 |

在 ExtensionPrimitives.register() 增加 r.add("deadzone", 2, a -> ...)，Python 中增加 implementations={"java":"deadzone"}。通过 build-runtime java 重新编译，无需改变 JPype 适配器。

```bash
python -m lgp build-runtime java --build-dir build/java --tests
java -cp build/java/classes org.modernlgp.runtime.TestExtensionPrimitives
```

Java 使用 javac --release 17。JPype 在同一进程内调用 Java double[][] 和 int[][]；它们仍有数据转换成本。JVM 只启动一次，各实例使用独立 URLClassLoader，防止自定义源码与其他实例的同名 Java 类串用。close() 关闭类加载器资源，不关闭共享 JVM。

原生源码随 wheel 安装，学习测试保存在项目源码包中。build-runtime --tests 需要源码项目；在其他位置构建原生测试时，build_cpp/build_java 可传 test_dir，CMake 可传 LGP_TEST_SOURCE_DIR。正常使用已安装 wheel 不需要测试源码。

## Python 类型扩展

Python 后端支持验证器认可的自定义向量或图像。每个样本创建独立状态，默认对象深复制；外部输入对象本身不会被自动深复制，纯原语应避免原地修改它。

原生后端当前要求全标量布局和纯操作，缺少原生实现不会静默切回 Python。

## 后端插件接口

能力描述是 RuntimeCapabilities，包括 backend、scalar_only、pure_only 和 batch。它是供调用方读取的声明，具体后端仍需实现正确校验。

```python
from lgp.runtime import RuntimeBase, RuntimeCapabilities, register_backend

class MyRuntime(RuntimeBase):
    capabilities = RuntimeCapabilities("my_backend")
    def run_many(self, individual, inputs):
        self._check_program(individual)
        # 校验输入、执行并返回 tuple[tuple[...], ...]
        ...

register_backend("my_backend", MyRuntime)
```

这个例子只说明接入约定，完整可运行测试使用 PythonInterpreter 注册别名。进化器通过结构化接口使用 runtime，不需要知道后端类型。

## 错误与资源

BackendUnavailable 是工具链缺失；NativeCompilationError 是编译配置或源码错误；UnsupportedPrimitive 是缺少实现或参数数量不一致。三者都明确抛出。数值溢出转换为 ArithmeticError，可由 Evaluator 赋予惩罚；无法分类的原生运行错误为 NativeExecutionError。

timeout 只限制 CMake/javac 编译进程，不中断进程内数值函数。cache_size 分别限制准备后的句柄缓存及兼容编码缓存，不是动态库卸载策略。auto_build=False 只加载 build_dir 产物，不触发构建；默认 True 保留自动构建。close() 不删除用户指定 build_dir；临时目录自动释放。已加载 C++ 库和 Java 类可能继续占用进程资源。

CMake 可选择 Linux、macOS 或 Windows 的编译器，但本次本地验证为 Linux，其他系统由使用者或后续 CI 验证。
