# AIM 2627 Python Coursework —— 哨兵 Sentry 控制模块

> **全部题目、规范、评分、提交见 [题面.pdf](题面.pdf)。** 本 README 只讲怎么把环境跑起来；没在这里出现的规格细节，一律以题面为准。

## 1. 环境要求

- Python 3.8+，仅标准库（不允许第三方运行时依赖）；
- 开发工具只需 `pytest`（测试）与 `autopep8`（风格，CI 会检查）；
- VS Code 打开仓库会推荐安装 `ms-python.autopep8` 插件（`.vscode/extensions.json`），保存即格式化即可过风格检查。

## 2. 快速开始

```bash
# 1. 用 GitHub 的 Use this template 创建你自己的仓库，然后 clone
git clone https://github.com/<你的用户名>/<你的仓库>.git
cd <你的仓库>   # 直接在 main 分支上开发

# 创建虚拟环境

python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

# 2. 装依赖
python -m pip install pytest autopep8

# 3. 启用 AI 会话归档钩子（课程要求，见下方第 3 节）
python -m pip install 'agent-session-commit[pre-commit]==0.1.3' -i https://pypi.org/simple
agent-session-commit install --pre-commit   # 交互选择你的 AI 助手与会话目录

# 4. 跑测试（刚到手：全部 skip，CI 是绿的）
python -m pytest

# 5. 看演示
python main.py

# 6. 打开 题面.pdf 读题，开始实现 src/main/__init__.py 里的 TODO
```

## 3. AI 会话归档（pre-commit）

本课程允许使用 AI，提交的 commit 需要携带 AI 会话归档作为透明化记录：每次 `git commit` 后，钩子会把新增会话自动 amend 进同一个提交（`.agent-sessions/bundles/`），不产生额外的归档提交。支持 Claude Code、OpenAI Codex CLI、GitHub Copilot CLI、Qoder、ZCode、Trae、Tencent CodeBuddy 等（完整名单见 [AgentLedger](https://github.com/Gentle-Lijie/AgentLedger)）。

- 配置是仓库本地的：每个 clone 运行一次 `agent-session-commit install --pre-commit`，方向键选择 agent、确认其会话目录即可；
- 不想用 TUI 可手动配置：`git config --local agent-session.agent claude`、`git config --local agent-session.source "<会话目录>"`，然后 `python -m pip install 'pre-commit>=3.2.0' && pre-commit install`；
- 归档是普通 Git 内容且会推送到公开仓库——不要在 AI 会话里粘贴令牌等敏感信息；
- 换了 agent 或目录就重跑一次安装命令；卸载：从 `.pre-commit-config.yaml` 移除该条目后重跑 `pre-commit install`。

## 4. 本地开发循环

- **写代码**：全部作业在 `src/main/__init__.py`，按题面各题规范补全每个标有 TODO 的函数；注释里标注了对应的题面主题，推荐顺序 Q1 → Q6。
- **跑测试**：`python -m pytest` —— 可见测试是规格书的一部分，未实现的函数自动 skip，实现一个、对应测试亮一个。本地全绿 ≠ 满分（见题面）。
- **看演示**：`python main.py`（等价于 `PYTHONPATH=src python -m main`），随实现进度逐段点亮，不进测试。
- **Q6 自测**：`python tools/run_seeds.py --q6`（200 张固定地图统计），单 seed 渲染 `python tools/run_seeds.py --q6 --seed <N> --render`，Bonus 模式 `python tools/run_seeds.py --bonus`。

## 5. 仓库结构（哪些能改）

| 路径 | 说明 | 能否修改 |
|---|---|---|
| `src/main/__init__.py` | 你的全部作业（TODO 所在） | ✅ |
| `README.md` | 仅末尾两个"你来写"小节 | ✅ |
| `题面.pdf` | 题面（唯一规格说明） | ❌ 勿改 |
| `src/main/legacy_patrol.py` | Q7 模块（与主体同步发布，修复其缺陷） | Q7 时 ✅ |
| `.pre-commit-config.yaml` | AI 会话归档钩子配置 | ❌ 勿改 |
| `src/tests/`、`tools/`、`.github/`、`conftest.py`、`pytest.ini`、`main.py` | 测试与基础设施 | ❌ 勿改 |

CI 只允许修改 `src/main/**`、`README.md` 与 `.agent-sessions/**`（AI 会话归档）——其余文件改了直接红；autopep8 `--diff` 非空即败。提交方式（push、问卷、commit 粒度）见题面"提交与验收"一节。

## 6. Q1–Q6 设计说明（你来写）

实现只使用 Python 标准库，沿用骨架提供的接口。实现过程参考了 AI 的解释与可见测试，
本节记录当前代码的设计和边界选择；相关 AI 会话由既有钩子随提交归档。

### Q1：血量计算与报告格式

`hp_ratio` 在 `max_hp <= 0` 时返回 0，否则计算百分比，使用 Python 的 `round`
规则得到整数，再将结果限制到 0–100。`status_report` 复用该函数，避免重复血量计算。
电量不超过 20 时为 `LOW`，不超过 50 时为 `WARNING`，更高时为 `OK`。
报告使用 f-string 控制字段宽度与对齐，返回单行字符串。

### Q2：混合日志解析与统一累计

每行先清理首尾空白，跳过非字符串、空行和注释。JSON 行通过 `json.loads` 读取，
校验部位与严格正整数伤害；布尔值、浮点数和数字字符串不作为合法伤害。
带 `id` 的记录在字段校验通过后才登记到 `seen_ids`，重复记录跳过；
无法作为集合键处理的 `id` 按脏行跳过，无 `id` 的记录独立统计。

传感器行按逗号分段，用正则表达式检查每段的部位字母和数字，再检查伤害大于 0。
整行先保存到临时事件列表，任一段非法就清空列表，保证脏行不会留下部分累计结果。
两种格式最终都转换为 `(armor, damage)` 事件，使用同一段代码更新三部位累计伤害。

平均值以有效事件数量为分母：每个传感器段计一次，每条有效 JSON 记录计一次，
最后使用 `round(total / event_count, 2)`。没有事件时，`most_hit` 为 `None`，
平均值为 `0.0`；伤害并列时按 `front`、`left`、`right` 的顺序选择第一个部位。

### Q3：位置校验、碰撞与电量

位置 setter 只接受长度为 2 的 tuple 或 list，复用 `_clamp_cell` 将元素转为整数、
限制在地图范围内，并以 tuple 存储。障碍位置被拒绝，校验成功后才更新当前位置。
新增异常提示使用英文，异常类型仍按输入类型与障碍情况分别为 `TypeError`、`ValueError`。

`move_forward` 在电量耗尽时直接返回当前位置；有电时，每次前进尝试消耗 1 单位电量。
障碍和地图边界都通过 `is_blocked` 判断，碰撞只增加计数，保留位置与朝向。
左右转使用单位方向向量的 90° 旋转公式，原地转向且不消耗电量。

### Q4：单步贪心导航

根据目标与当前位置的坐标差生成朝目标移动的候选方向。对于整数网格坐标，
这些方向每走一格都会使曼哈顿距离减少 1。优先考虑绝对坐标差更大的轴，
差相等时先考虑水平方向，再考虑竖直方向。

按优先顺序排除障碍，返回第一个可用的 `Facing`；没有候选时返回原朝向。
函数只选择方向，实际位移、电量和地图边界由载体处理。

### Q5：按优先级求值的纯函数决策

`decide` 先校验 sensor 字段是否齐全、state 是否为枚举成员，并对 tuple 或 list 帧历史检查长度是否为 1–6。
契约外输入抛出 `ValueError`；已有字段的异常取值则按当前实现进行防御式规范化。
例如无法计算的血量按 0% 处理，非整数敌距按无穷远处理，
非字符串机型使用步兵标识，未知字符串机型沿用步兵动作分支。

规则按 R1–R7 的优先级执行，首条命中立即返回，确保低血量撤退优先于交火。
R4 和 R6 的交火动作共用内部函数 `engage_action`，保持距离与机型判定一致。
帧历史末位表示当前检测，末两位都为真才确认敌情；交火中丢失当前帧时，
前一帧仍可见则保持交火，否则切入扫描。

函数只返回动作和新状态，不修改输入 sensor。保留 `heat` 接口，决策严格遵循规则表，
没有额外添加会改变优先级或返回动作的热量条件。

### Q6：贪心巡逻与 BFS 脱困

每轮读取载体状态，用 Q4 获取方向，对齐朝向后调用 `move_forward`。
贪心方向被障碍或边界阻挡，或者下一格不能严格缩短曼哈顿距离时，启动 BFS 脱困。
BFS 使用 `deque` 队列按距离扩展格子，使用父节点记录重建从当前位置到目标的路线；
搜索通过 `is_blocked` 限制在地图内，并跳过已发现格子，避免重复搜索。

脱困路线找到后缓存为方向队列，每轮取出一个方向，直到走到目标，避免再次陷入同一死角。
BFS 只在需要脱困时启动。每次搜索的时间和空间均与地图格子数量同阶，
搜索过程只规划路线，实际移动仍由载体执行。

主循环在抵达目标、步数达到 `max_steps` 或电量耗尽时结束；
如果 BFS 确认目标不可达，也会结束并报告失败。
`steps` 统计前进尝试次数，与 seed 工具一致，转向不单独累计；
`visited_count` 统计不同位置数量，包含初始位置。
碰撞次数读取载体计数，`found_enemy` 与 `success` 都使用最终是否抵达目标的结果。
`report_to_json` 使用 `sort_keys=True` 固定键顺序，生成确定性的 JSON 字符串。

任务自测使用工具提供的 1000 单位电量。当前 200 张固定地图的本地结果如下：

| 指标 | 本地结果 | 题面阈值 |
|---|---|---|
| 成功率 | 100% | ≥ 92% |
| 平均碰撞 | 0.00 | ≤ 1.5 |
| 成功案例平均步数 / BFS 最短路 | 1.03 | ≤ 1.35 |

全仓库可见测试为 34 项通过、1 项跳过；跳过项对应尚未实现的 Bonus
`bfs_path_length` 接口。上表记录本地固定地图的结果，隐藏测试由批改端验收。

## 7. Q7 缺陷定位与修复（你来写）

本次使用 AI 协助复现、定位和修复。验收依据是题面 Q7 和
`src/main/legacy_patrol.py` 各函数的 docstring。修复前，该文件的本地提交历史只有
`Initial commit`，没有题面所述的历史 `fix` 提交，因此以当前实现与契约的差异为准。

修复前运行可见测试，先排除可能卡死的阈值测试：

```powershell
.\.venv\Scripts\python.exe -m pytest src/tests/test_legacy.py -q -k "not sim_stops_at_threshold"
```

结果为 5 项失败、1 项通过、1 项排除。对可能卡死的调用使用 Python 行级跟踪，
限定模拟函数最多执行 100 次行事件，确认循环无法按预期终止，避免无限积累 `trace`。

六处主体缺陷及修复如下：

1. **路线长度单位错误。** `segment_length_cm` 返回厘米，但
   `total_route_meters` 直接累计后作为米返回。路线 `(0, 0) → (3, 0) → (3, 4)`
   得到 700，契约要求 7 米。改为先累计厘米，再除以 100；使用普通除法以保留小数米。
2. **无正样本时没有处理空基线。** `first_positive` 返回 `None` 后，
   `calibrate` 仍执行减法。`calibrate([-1, -2])` 因此抛出 `TypeError`。
   增加 `baseline is None` 判断，按契约返回 0。
3. **事件编号上界遗漏。** “id 不超过 max_id”要求包含相等情况，原实现使用 `<`。
   改为 `<=`，使 `id == max_id` 的事件也参与统计。
4. **默认日志历史被共用。** `history=[]` 在函数定义时只创建一次，连续调用
   `log("a")`、`log("b")` 得到共用列表。改为默认值 `None`，在每次未提供历史时
   创建新列表；显式传入历史列表时仍追加到该列表。
5. **模拟停止条件反向。** 原实现体力 `> 20` 就停止，导致体力充足时只执行一轮。
   改为在任一轮结束后体力 `<= 20` 时停止，包含恰好等于 20 的边界。
6. **模拟轮号没有递增。** `round_` 一直为 0，轮数限制和第 4 轮起的额外消耗
   都无法正常生效。每轮记录轨迹后增加 `round_ += 1`，保证轮号推进和有限轮数。

两组互相遮蔽的问题也分别通过仅修改一个条件的内存实验确认：

- 原编号上界错误会跳过 `id == max_id` 的事件；仅改成 `<=` 后，
  该事件的负样本会触发空基线减法异常，暴露第 2 处缺陷。
- 原停止条件会让高体力模拟提前结束，掩盖轮号不递增；仅修正停止条件后，
  `run_legacy_sim(2, 100)` 执行了 10 轮，轨迹轮号全为 0，暴露第 6 处缺陷。

此外，按 `parse_event` 的“脏行返回 None，不得抛异常”契约补充防御处理：
非字符串输入直接跳过；数字转换失败时返回 `None`。例如 `None` 原先触发
`AttributeError`，`"MOVE,²"` 虽通过 `isdigit()` 却在 `int()` 处触发 `ValueError`。
这项补充独立于上面的六处主体缺陷。

修复后全仓库可见测试为 34 项通过、1 项跳过；跳过项是尚未实现的 Bonus。
另用 260 组轮数与初始体力组合对照契约验证模拟结果，并检查小数米、
无正样本、编号上界、日志列表独立性和脏行解析。autopep8 检查差异为空。

复核命令：

```powershell
.\.venv\Scripts\python.exe -m pytest
.\.venv\Scripts\python.exe -m autopep8 --diff --recursive src/main
```
