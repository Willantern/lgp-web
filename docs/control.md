# 控制任务

控制策略把环境观测转换为动作。Gymnasium 是常见的环境接口，提供 `reset()` 和 `step()`，返回奖励以及终止标记。

```bash
python -m pip install -e '.[control]'
python examples/control.py
```

示例使用 CartPole，只评价一个手写、未训练的候选程序，不声称超越强化学习。

## 环境协议

`ControlTask(env_factory, seeds, max_steps, input_registers)` 为每个种子创建并最终关闭一个环境。

- 连续动作空间：输出数需要与动作维度匹配，输出按环境边界裁剪。
- 离散动作空间：每个动作需要一个输出分数，选择分数最大的动作；相同分数按 NumPy argmax 的规则选择最前的索引。
- 观测目前要求可展平为数值向量，长度与输入寄存器数量一致。
- 环境返回 `terminated` 或 `truncated` 时停止当前 episode。

返回值为多种子平均奖励的负数，因为进化器统一最小化目标。

## 接入进化

将示例中的 `ControlTask` 作为 `Evaluator(runtime, task)` 的目标函数即可。根据观测维度和动作数量创建寄存器，使用相同的 ProgramFactory 与 evolve。

控制任务建议配置 `target_fitness=None` 或设定负奖励阈值；回归默认的 `1e-12` 通常不适合控制任务。

## 目前的边界

策略是无跨时间状态的映射。积分项或低通滤波的历史量需要显式放入观测。没有改写仿真器时间步，没有高频控制协议，也没有自动把评价次数转换为 RL 环境交互预算。

MuJoCo、图像观测、Dict 观测、多智能体及特定环境的训练协议需要额外任务适配。基础框架已提供接入接口，没有声称完成这些应用的基准实验。
