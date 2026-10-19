# Modern LGP

线性遗传编程（Linear Genetic Programming，LGP）通过进化搜索一段操作寄存器的程序。

0.4 提供 Python、SWIG C++ 和 JPype Java 三种执行器。C++ 使用独立头文件、实现文件、SWIG 接口和 CMake；Java 使用独立 package。新增完整 Task 评分协议，C++/Java 可以在自己的仿真环境内反复执行准备后的程序，再向 Python 返回适应度。见 [完整任务接入](tasks.md)。

| 层 | 职责 |
| --- | --- |
| program | 程序表示、类型与寄存器规则 |
| runtime | 执行程序、原生构建、后端扩展 |
| fitness | 调用 Task、上下文缓存和数值失败处理 |
| evolution | 初始化、选择、变异、交叉与搜索 |
| tasks | 完整评分过程、数据和环境 |

从 [安装与开始](getting-started.md) 运行示例，沿 [学习路线](learning.md) 阅读逐文件测试。后端维护和新增原语见 [runtime 文档](runtimes.md)。参考仓库取舍见 [设计审查](design-review.md)。
