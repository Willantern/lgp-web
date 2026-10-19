# 0.4 架构与执行语义

三个核心角色：

| 角色 | 职责 |
| --- | --- |
| Individual / ProgramSpec | Individual 是进化基因型；ProgramSpec 是带寄存器布局的完整标量执行描述 |
| Task | 拥有完整评分过程，可调用 runtime，也可直接调用外部原生环境 |
| Evaluator | 调用 Task、缓存、计数、处理数值失败，并返回附带适应度的新 Individual |

evolve 仅依赖 Evaluator，不依赖语言桥接或具体任务。runtime 是 Task 可以选择使用的程序执行工具。

## 完整原生评分

原生任务的实际调用顺序：

1. Python Task 首次创建原生任务对象，固定数据或仿真配置只跨桥传一次。
2. runtime.native_program(individual) 校验并绑定 ProgramSpec；有界缓存复用同一描述的 PreparedProgram。
3. 原生构造函数解析名称、检查参数数量/索引/只读权限、预分配执行寄存器与每条指令参数缓冲。
4. Task 调用原生 score(handle) 一次；原生环境内部反复调用 handle.execute(observation)。
5. 原生返回一个评分，Python 包装成 TaskResult。进化、选择与变异仍在 Python。

PreparedProgram 不生成或编译候选源码。CMake/javac 编译框架、原语和任务实现；候选仅是解释器数据。run_many 也使用准备后的句柄批量执行；原有原生 evaluate 函数为 0.3 直接调用兼容保留。

## 寄存器与重置

只读且无默认值的寄存器是外部输入，按索引升序排列；只读且有默认值的是常量。可写标量无默认值时初值为零。ProgramSpec 保存全部标量初值和权限。

每次 execute 都恢复完整寄存器初值，再写入本次输入。因此程序没有跨样本或跨仿真步记忆。仿真环境的位置/速度由 Task 保存；每回合自行重置。需要有记忆的策略时应新增明确的状态协议，不能依赖未清空的寄存器。

Python 执行支持自定义类型验证器；非标量读取前必须初始化。原生后端只支持纯标量。原生直接调用也会检查尺寸、索引、只读权限、参数数量和有限数值。

## 缓存的两个层次

Evaluator 缓存任务评分，只在 Task.cache_key 非 None 时默认开启。键必须代表固定数据、种子、环境设置和实现版本。上下文改变会清空旧分数；评价中途改变上下文则报错。没有种子控制的随机任务默认不缓存，原语纯度无法证明环境确定性。

NativeInterpreter 的有界缓存保存 PreparedProgram，与适应度缓存独立。ProgramSpec 包含布局和初值，因此同一 Individual 绑定不同布局不会误用旧执行句柄。evolve 开始时重置评分缓存及计数；runtime 的句柄缓存可继续复用。

## 资源与扩展

推荐显式 build-runtime，再使用 auto_build=False。auto_build=True 保留首次使用自动构建的便利。prepare() 在单个实例中只加载一次。用户 build_dir 不会在 close() 时删除；临时目录会清理。Java 不关闭共享 JVM。

PreparedProgram 持有可变工作区；每个并发 worker 使用独立 runtime 和 Task，不共享句柄。单进程桥接不提供硬超时、崩溃隔离或取消能力。

新增任务通常只增加任务类与小型 Python Task，不增加第二套进化器、任务路由器或评分后端体系。具体步骤见 [完整任务接入](tasks.md)。

offspring() 清除适应度，with_fitness() 返回新对象。effective_indices/prune 仍是独立纯程序分析；精简可能改变无效指令引发的异常，因此评价不自动精简。
