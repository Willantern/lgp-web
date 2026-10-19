# API 参考

## 程序层

| 接口 | 职责 |
| --- | --- |
| PrimitiveSet.register(name,inputs,output,func,category,implementations=...,pure=True) | 注册原语和原生名称映射 |
| PrimitiveSet.freeze() | 稳定操作编号 |
| PrimitiveSet.implementation(opcode,backend) | 查找原生实现名称 |
| PrimitiveSet.native_implementations(opcode) | 返回名称映射副本 |
| Instruction(opcode,dst,src) | 不可变指令 |
| Individual(instructions,outputs,fitness=None) | 进化指令与输出基因型 |
| ProgramSpec.bind(individual,primitives,registers,backend) | 完整标量执行描述，包括初值与读写布局 |
| ProgramSpec.to_json / from_json | 执行描述往返，不等于 Individual JSON |
| Individual.validate(primitives,registers) | 静态类型与初始化检查 |
| Individual.offspring(instructions) | 创建无适应度子代 |
| Individual.with_fitness(value) | 附带有限适应度的新对象 |
| RegisterFile(types,readonly=(),validators=None,defaults=None) | 寄存器布局和状态 |
| effective_indices / prune | 输出依赖分析与精简 |
| save_program / load_program | schema 2 写入、schema 1/2 读取 |

## 执行与扩展

| 接口 | 职责 |
| --- | --- |
| RuntimeBase | 继承并实现 run_many()，复用生命周期与输入检查 |
| RuntimeCapabilities | 描述 backend、scalar_only、pure_only、batch |
| PythonInterpreter(primitives,registers) | 通用 Python 执行 |
| CppInterpreter(primitives,registers,cmake="cmake",swig="swig",compiler=None,...) | CMake/SWIG 执行 |
| JavaInterpreter(primitives,registers,javac="javac",jvm_path=None,classpath=(),...) | JPype 执行，可加入第三方 jar/类目录 |
| runtime.run(individual,inputs) | 返回一组输出元组 |
| runtime.run_many(individual,inputs) | 返回批量元组 |
| native.prepare() | 显式构建和加载 |
| native.native_program(individual) | 准备/复用原生句柄，解析名称与分配工作区 |
| native.prepare_spec(spec) | 从完整标量描述准备/复用原生句柄 |
| native.native_class(name) | 加载用户暴露的原生任务类；Java 传完整限定名 |
| native.score_task(task,individual) | 调用任意含 score(handle) 的原生任务并映射异常 |
| runtime.close() | 关闭实例 |
| register_backend(name,factory,replace=False) | 注册自定义适配器 |
| available_backends() | 已注册名称，未表示可选工具链已安装 |
| make_interpreter(name,primitives,registers,...) | 按名称创建 |
| build_cpp / build_java | 从 source_dir 构建到 build_dir |
| source_directory("cpp"或"java") | 查找安装包中的原生源码 |

原生构造参数还包括 build_dir=None、source_dir=None、timeout=120、cache_size=64、auto_build=True。auto_build=False 要求显式 build_dir，只加载已有产物。timeout 只限制构建进程。详见 [runtime 文档](runtimes.md)。

## 优化与任务

| 接口 | 职责 |
| --- | --- |
| EvolutionConfig | 进化参数与校验 |
| ProgramFactory | 类型约束的程序初始化 |
| mutate / crossover | 不修改父代的变异和片段交叉 |
| tournament / rank_selection | 按最小化目标选择 |
| Population | 种群及 best |
| Evaluator(task,cache=None,failure_fitness=1e100) | 独立于 runtime 的评分、缓存和数值失败处理 |
| Task.evaluate(individual) | 返回 TaskResult 的结构化协议 |
| TaskResult(fitness,metrics={}) | 有限适应度与只读工作计数 |
| RuntimeTask(runtime,objective,context_key=None) | 包装旧回调；默认不缓存 |
| tasks.native_tasks.RegressionTask(runtime,data) | 完整任务评分，按后端选择原生或 Python |
| tasks.native_tasks.PointMassTask(runtime,positions=...,steps=100,dt=0.05) | 原生完整控制仿真及 Python 对照 |
| evolve(factory,evaluator,config,initial=None) | 完整搜索 |
| SymbolicRegression | 多输出均方误差 |
| ControlTask | Gymnasium 协议和平均奖励负值 |
| make_rng | 局部随机数 |
| save_run | 程序、配置和每代日志 |

EvolutionResult 保存 best、population、history、evaluations、cache_hits、failures、stop_reason。
