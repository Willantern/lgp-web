# 原语与类型扩展

原语是一种程序操作。PrimitiveSet 保存 Python 函数、输入输出类型，以及指向原生实现的名称。Python 名称、Python opcode 和原生名称可以不同。

## 注册一个增益操作

```python
p.register(
    name="gain",
    inputs=("scalar","scalar"),
    output="scalar",
    func=lambda error,gain: error*gain,
    category="control",
    implementations={"cpp":"gain","java":"gain"},
)
p.freeze()
```

C++ gain 已在 _native/cpp/src/extensions.cpp 注册，Java gain 已在 ExtensionPrimitives.java 注册。这两个函数执行相同数学运算，独立测试和跨后端测试用于验证一致性。

只写 implementations 不会自动创造 C++/Java 函数。先在对应原生注册表中提供函数，再构建后端；参数数量在执行前比对。新增过程见 [runtime 文档](runtimes.md)。

## Python 专用类型

```python
import numpy as np
from lgp import PrimitiveSet,RegisterFile
p = PrimitiveSet()
p.register("mean",("image",),"scalar",lambda x:float(np.mean(x)),category="vision")
r = RegisterFile(("image","scalar"),readonly=(0,),
                 validators={"image":lambda x:isinstance(x,np.ndarray) and x.ndim==2})
```

这类程序通过 PythonInterpreter 执行。类型名本身不描述通道、尺寸或 dtype，验证器负责这些规则。原生后端明确限制为纯标量布局。

## 数值与参数

纯原语指相同输入有相同输出，并且没有修改外部状态的副作用。pure 标记不会自动证明代码满足定义。当前 run() 每次清空寄存器，积分器、延迟和滤波历史应显式传入。

gain 等可调值通过输入或常量寄存器传递，不放在会变化的闭包中。进化器可以选择常量寄存器，尚未实现连续参数优化。

内置 div 在 abs(b)>1e-12 时计算 a/b，否则返回 a。sat 限制到 [-1,1]，其他基本算术不做隐式裁剪。三种后端逐步检查有限数值。

## 从 0.2 迁移

旧代码：

```python
expressions={"cpp":"({0}*{1})","java":"({0}*{1})"}
```

新代码：

```python
implementations={"cpp":"gain","java":"gain"}
```

数学实现移动到 C++/Java 源码。旧 runtime.compiler 模块已移除，不再保留运行时拼接原生源码的路径。任意 Python 函数不会被翻译成原生代码。新程序 JSON 使用 schema 2 保存名称映射；旧 schema 1 仍按原签名规则读取。
