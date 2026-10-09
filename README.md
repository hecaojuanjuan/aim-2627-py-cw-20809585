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

## 6. Q7 缺陷定位与修复（你来写）

本次使用 AI 协助复现、定位和修复。验收依据是题面 Q7 和
`src/main/legacy_patrol.py` 各函数的 docstring。该文件的本地提交历史只有
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
