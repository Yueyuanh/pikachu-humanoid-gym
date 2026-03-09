# `ref_state.py` 使用说明

## 功能
`ref_state.py` 用于可视化参考步态（reference gait）：

1. `sim` 模式：在 Isaac Gym 中加载任务对应 URDF，并按参考步态驱动关节运动。  
2. `plot` 模式：绘制参考相位、关节参考偏移、`ref_action` 曲线。  
3. `both` 模式：先仿真播放，再绘图。

脚本会根据 `--task` 自动读取对应配置和 URDF 路径，不需要在脚本内手工改任务映射。

---

## 脚本位置
- [ref_state.py](/home/finnox/Pikachu/pikachu-humanoid-gym/humanoid/scripts/ref_state.py)

---

## 运行前准备
建议在训练环境中运行（例如 `rl_amp`）：

```bash
cd /home/finnox/Pikachu/pikachu-humanoid-gym/humanoid
python scripts/ref_state.py --help
```

需要环境中可用：
- `isaacgym`
- `torch`
- `numpy`
- `matplotlib`（仅 `plot` 或 `both` 模式需要）

---

## 常用命令

1. 默认仿真可视化（推荐）
```bash
python scripts/ref_state.py --task Pikachu_V025
```

2. 仅曲线图
```bash
python scripts/ref_state.py --task Pikachu_V025 --mode plot
```

3. 仿真+曲线
```bash
python scripts/ref_state.py --task Pikachu_V025 --mode both
```

4. 覆盖步态参数
```bash
python scripts/ref_state.py \
  --task Pikachu_V025 \
  --cycle_time 0.52 \
  --target_joint_pos_scale 0.025
```

5. 修改仿真基座抬高高度
```bash
python scripts/ref_state.py --task Pikachu_V025 --base_lift 0.7
```

6. 取消固定基座（默认是固定）
```bash
python scripts/ref_state.py --task Pikachu_V025 --free_base
```

7. 只播放一轮参考轨迹（默认会循环）
```bash
python scripts/ref_state.py --task Pikachu_V025 --no_loop
```

---

## 参数说明

- `--task`：任务名（例如 `Pikachu_V025`、`Pikachu_V01`、`humanoid_ppo`）。  
- `--seconds`：可视化总时长（秒）。  
- `--dt`：参考步态计算步长；默认 `cfg.control.decimation * cfg.sim.dt`。  
- `--cycle_time`：覆盖 `cfg.rewards.cycle_time`。  
- `--target_joint_pos_scale`：覆盖 `cfg.rewards.target_joint_pos_scale`。  
- `--mode {sim,plot,both}`：可视化模式。  
- `--headless`：`sim` 模式下不创建 viewer。  
- `--base_lift`：仿真时基座额外抬高（米）。  
- `--free_base`：关闭固定基座（默认固定基座）。
- `--no_loop`：只播放一次轨迹（默认循环播放直到关闭窗口）。

---

## 仿真控制说明
在 `sim` 模式下，脚本默认：

1. 固定 `base_link`（便于观察纯关节参考动作）。  
2. 抬高基座（默认 `+0.5m`）避免碰撞遮挡。  
3. 使用 `DOF_MODE_POS` + 任务配置中的 `stiffness/damping` 做 PD 位置控制。  

---

## 参考步态计算逻辑
脚本复现了环境中的 `compute_ref_state` 逻辑：

1. 用 `phase = t / cycle_time`，再算 `sin(2πphase)`。  
2. 左腿只保留负半波，右腿只保留正半波。  
3. `hip/ankle` 用 `scale_1`，`knee` 用 `2*scale_1`。  
4. 双支撑区间 `abs(sin) < 0.1` 时，将 `ref_dof_pos` 置零。  
5. `ref_action = 2 * ref_dof_pos`。  

---

## 常见问题

1. `ImportError: PyTorch was imported before isaacgym modules`
- 原因：导入顺序问题。  
- 当前脚本已按可用顺序处理；请直接运行脚本，不要在外部预先导入冲突模块。

2. `TypeError ... contact_collection`
- 原因：IsaacGym 绑定要求枚举类型而不是裸整数。  
- 当前脚本已做兼容映射（`0/1/2` -> 对应枚举）。

3. 没有弹出曲线图
- 检查是否安装 `matplotlib`。  
- 若在无图形环境，先用 `--mode sim --headless` 验证仿真侧逻辑。

---

## 建议工作流

1. 先用 `--mode sim` 看关节参考动作是否符合预期。  
2. 再用 `--mode plot` 对比相位与关节轨迹形状。  
3. 调 `--cycle_time` / `--target_joint_pos_scale` 后重复以上步骤，再进入训练。  
