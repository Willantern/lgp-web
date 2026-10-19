# 安装与开始

要求 Python 3.10+。在项目根目录执行：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -e .
python examples/basic_execution.py
python -m lgp regress --seed 42 --output runs/demo
python -m lgp predict runs/demo/best.json --x -1 0 0.5 1
```

-e 是可编辑安装，修改 Python 源码会被当前环境使用。Windows 环境激活命令为 .venv\\Scripts\\activate。

## 安装后端和测试依赖

```bash
python -m pip install -e '.[dev]'
python -m lgp doctor
python -m pytest -q
python -m mkdocs serve
```

基础包无第三方 Python 依赖，原生后端按需构建。C++ 需要 CMake、SWIG、C++17 编译器和 Python 开发头文件；Java 需要 JPype 和 JDK 17+。系统工具缺失时不会静默跳到其他执行器。

## 手动构建

```bash
python -m lgp build-runtime cpp --build-dir build/cpp --tests
ctest --test-dir build/cpp -C Release --output-on-failure
python -m lgp build-runtime java --build-dir build/java --tests
python -m lgp regress --backend cpp --build-dir build/cpp --output runs/cpp
python -m lgp regress --backend java --build-dir build/java --output runs/java
```

后端构建详情、source_dir 和扩展方法见 [runtime 文档](runtimes.md)。

回归示例拟合 x*x+x，采用 [-1,1] 上 41 个均匀训练样本，是框架功能验证。每次运行保存 best.json、history.csv 和 run.json，记录配置、任务、评价次数、失败次数和每代结果。
