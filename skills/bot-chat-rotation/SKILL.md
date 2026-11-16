---
name: bot-chat-rotation
description: "归档 bot 的主对话，另开一条会读旧对话的新主对话。"
version: 0.1.0
author: <user>, Hermes Agent
license: MIT
platforms: [windows, macos, linux]
metadata:
  hermes:
    tags: [Bot-Mode, Session, Rotation, Handoff]
    related_skills: [hermes-desktop-session-relay, project-secretary]
---

# Bot 主对话轮换

把 bot 的主对话（Bot Chat）归档，另建一条同名的新的，并让新的第一件事就是读完旧的。走桌面后端，与 Bots 面板点击同一条路。

## When to Use

- 「把所有 bot 的主对话归档，开新的读旧的」「bot chat 轮换 / 重建 / 重开一条」。
- 某条线的主对话压满了，要换一条干净的接着跑。
- 不是这个：单会话 `/compact`（不换会话）、普通会话归档（右键归档即可）。

## 本质

- **默认只动脚本自己所在的 profile**（发给哪个 bot，就重置哪个）。每个 bot 的 skills 目录各有一份脚本，不带参数运行时目标就是它自己（HERMES_HOME，回退看安装路径）；全机轮换必须显式 `--all`。
- **Bot Chat 的身份 = (profile, 标题正好是 `Bot Chat`)**，没有 id 指针。用 `session.create {title:'Bot Chat', hidden:true, follow_profile_config:true}` 建，随后 `session.title` 立刻落标题——行是懒建的，不落标题就会开出第二条。
- **顺序：先归档旧行，再给新行落标题。** 标题事务会把「已归档且 hidden」的旧行 title 置 NULL，注册名让给新行。所以归档后旧对话**只能按 id 读，按标题已找不到**——承接指令里必须写 id。
- **新会话的第一句就是承接指令**（`session_search(session_id=<旧 id>)`）。不投这句，新主对话就是空的。
- 桌面后端 = Electron 起的 `hermes serve`，端口与 token 每次现取。**`hermes -p X chat -q` 不算**：它建的会话不是 Bot Chat，点 bot 不会落到它上面。

- **承接靠线状态卡，不靠通读旧对话**（2026-10-19 起；2026-11-08 简化；2026-11-16 再减负）：开场句让新会话依次读 `project-discipline` → `<WORKSPACE>/团队工作法.md` 〇、二两节 → `<WORKSPACE>/line_state/<profile>.md`；**线状态卡就是工作状态正本，照它认方向即可，不再要求读项目索引与禁用口径表**（旧模板第 4/5 步读 `CURRENT_INDEX.md`+`CALIBER_GUARDRAILS.md`，已删）——卡或卡内缺项、或卡的更新时间早于旧对话最后一条消息，才按 id 翻旧对话补缺，**读完直接干活，不再回「承接三答」、不向总管报到**（用户 11-08 令：流程减重）；没活只回一行「空闲＋想做什么」。轮换前先让旧会话刷新卡、删记忆里已被推翻的口径。开场句只放通用步骤，**不写某一条线的状态**（旧模板写死过一条线的速览，`--all` 时会塞给每个 bot）。
## 跑

```bash
# 用 Hermes venv 的 python 最好；系统 python 直接跑也行——脚本自己 re-exec 到 venv，找不到才报错。
python "<skill>/scripts/rotate_bot_chats.py" --dry-run         # 先看会动谁（默认：本 profile）
python "<skill>/scripts/rotate_bot_chats.py"                   # 轮换本 profile——发给哪个 bot 就重置哪个
python "<skill>/scripts/rotate_bot_chats.py" research steward  # 指定 profile
python "<skill>/scripts/rotate_bot_chats.py" --all             # 本机全部（显式全量）
```

脚本一条龙：找后端端口/token → `profiles.list` 取各 profile 的 `canonical_session` → `session.archive` → `session.create` → `session.title` → 回读 state.db（找得到就读，找不到只给 RPC 侧）核验 → `prompt.submit` 首句承接指令。首句模板在脚本顶部 `OPENING`，要改承接要求就改它。

## 装到别的机器 / 别的 bot

- 把整个 `bot-chat-rotation/` 目录拷到目标机任意 profile 的 `skills/<category>/` 下就能用；**profiles 互不继承**，要每个 bot 都能用就逐 profile 拷一份。
- 目标机前提：桌面应用（或任何 `hermes serve` 后端）在跑、脚本能连上它的 127.0.0.1 WS。脚本不写死 profile 名、端口、路径：bot 名单来自后端 roster，端口自己探；`HERMES_PYTHON` / `HERMES_DESKTOP_URL` / `HERMES_SESSION_TOKEN` 可覆盖探测结果。
- 后端在别的机器（SSH / 远程 connection）时，脚本要在**那台机器上**跑——它只走 127.0.0.1。

## 核验（不信 ACK）

脚本跑完自己打印核验：RPC 侧看 `session.list {title:'Bot Chat', include_hidden:true}` 只剩新行、`profiles.list.canonical_session` 指向新 id；本机找得到 state.db 就再直读两行（旧行 `archived=1 / title IS NULL`——`hidden` 可能留 0，别拿它当判据；新行 `title='Bot Chat' / hidden=1 / source='desktop'`）。读取进度看新行 `messages` 增长与 `session_turn_leases`（有 lease 才是在跑）——turn 由桌面后端持有，脚本断开也照跑。

## 坑

- Windows 上系统 `python` 自动 re-exec 可能静默返回 `rc=0` 而没有 dry-run/轮换输出；这不算成功。直接用已验证的 Hermes venv `Scripts/python.exe -u` 调同一脚本，看到目标和核验输出再判完成，不重复盲跑正式轮换。

- 动手前查 `session_turn_leases`：忙会话会被 `prompt.submit` 打断当前回合。
- WS 连接必须 `max_size=None`（服务端重放 `gateway.ready` 大帧，默认 1MB 上限会打死连接）。
- `prompt.submit` 用**运行态** session_id（`session.create` 返回的短 hex），存储 id 会 4001。
- 旧行标题被置空是**预期**行为，不是丢数据：消息全在，按 id 读得到。
- 回退：`session.archive {archived:false}` 解归档；要让旧行拿回 `Bot Chat` 标题，得先让新行让位。
- 只动 `Bot Chat`，不碰别的会话；不手改 state.db。
- **旧会话回合被打断（中途停止/出错）后轮换**：被打断回合的动作可能只跑了一半（写了件没写卡、核了一半下了结论）。交接卡里逐条记明「哪些动作可能未跑完、哪些结论未验收」，新会话承接先核盘上实况再续做；被打断回合的口头结论不当完成件。


---

## 合并补充（2026-11-08，从各副本备份恢复）

### 来自 backtest

- **轮换 ≠ 旧会话立即停跑**：旧会话在途的后台通知/长 turn（如作业守望通知唤醒的核验链）会照常跑完，轮换后两个会话的 turn 可短暂并行（各有 lease）。新会话读旧对话时先核旧会话尾部在途任务：让它自主收口、或明确接管；别双头重做同一件事。
- **长对话读法**：消息多、压缩轮次多时，`session_search` 的默认窗口不够——直读 state.db：按 session_id 过滤后看 `(active, compacted)` 分布，压缩摘要（正文以 `[CONTEXT COMPACTION` 开头）是最完整的阶段快照，再依次读活跃段的 user 消息全文与 assistant 文本；重复快照按内容去重；**副本行识别**：压缩轮次与接口切换（CLI↔desktop）都会重放历史，同一逻辑消息的副本行时间戳逐字段相同——按「时间戳+内容」去重，勿按行数计消息、勿误判为双头执行。

### 来自 eigen-zero

- 严格承接先查旧对话尾部是否有未落卡的新工单；旧会话写卡被拒不算已更新，必须按最后实际工单补齐优先级、未过闸项和下一步再轮换。遇 `stale_write_blocked` 优先定向 `patch`，不要重复同一全量覆盖。轮换已核验不等于新会话已承接完；只有新会话真实三答/回执落库才报承接完成。
- Windows 上系统 Python 自动 re-exec 可能返回 rc=0 却无任何输出；这不算已执行。直接用 `<HERMES_HOME>/hermes-agent/venv/Scripts/python.exe` 重跑 `--dry-run`，看到目标与旧 id 后再实际轮换。
- 轮换后 bot 互发消息若静默失败（ambiguous /「liveness unknown」）：见 `bot-delivery-troubleshoot` 技能（运行时 psutil 缺失的诊断 + 安全补投）。

### 来自 engineering

- 严格承接先查旧对话尾部是否有未落卡的新工单；旧会话写卡被拒不算已更新，必须按最后实际工单补齐优先级、未过闸项和下一步再轮换。遇 `stale_write_blocked` 优先定向 `patch`，不要重复同一全量覆盖。轮换已核验不等于新会话已承接完；只有新会话真实三答/回执落库才报承接完成。
- Windows 上系统 Python 自动 re-exec 可能返回 rc=0 却无任何输出；这不算已执行。直接用 `<HERMES_HOME>/hermes-agent/venv/Scripts/python.exe` 重跑 `--dry-run`，看到目标与旧 id 后再实际轮换。
- 轮换后 bot 互发消息若静默失败（ambiguous /「liveness unknown」）：见 `bot-delivery-troubleshoot` 技能（运行时 psutil 缺失的诊断 + 安全补投）。

### 来自 ops

Windows 上若系统 Python 跑脚本无输出却 rc=0，不能当成功：自动 re-exec 可能未实际完成。直接用已存在的 Hermes venv `Scripts/python.exe -u` 运行原脚本，再以脚本打印的 RPC/数据库核验结果判定；不要反复重试系统 Python。

### 来自 recon

- **系统 python 直跑会静默退出**（2026-10-09 本机实测：无输出、rc=0；`os.execv` 自举无可见效果）。用 Hermes venv 的 python 直跑：本机 `<HERMES_HOME>/hermes-agent/venv/Scripts/python.exe`（内含 websockets 15.0.1），或设 `HERMES_PYTHON=<venv-python>`。
- Windows 上系统 python 直跑时输出会「晚到」：脚本 os.execv re-exec 后父进程即刻以 rc=0 退出，子进程输出晚于命令返回——终端里看起来「无输出、rc=0」。要看输出：用 venv python 直跑（Windows 路径 `<hermes-home>/hermes-agent/venv/Scripts/python.exe`），或 `> 文件 2>&1` 后 sleep 再读文件。**dry-run「无输出」先查输出文件/DB，切勿重跑**——重跑会二次轮换。

### 来自 research

- **承接记录不能只读自己那条线**：若旧对话里几乎只有「收规矩/收通知」而无任务（本线曾 87 条里 85 条如此），必须交叉读队友的现行 Bot Chat（`<profile>/state.db` 里 `title='Bot Chat'` 那行）确认主线在哪、本岗有没有被派工——否则承接记录会把别线的待办当自己的，或漏掉本线的挂起活。

### 来自 secretary

- **何时轮（用户 10-19 定）**：看上一轮 API 的 `in=`（当前上下文 token），**超 25 万就轮**。不按条数。忙线（该 Bot Chat 有 turn lease）跳过。脚本 `scripts/rotate_if_over.py`；Hermes cron `no_agent` 每小时跑一次，stdout 有字才投。
- **没有 `--help`**：脚本只认 `--dry-run` / `--all` / `--no-prompt`；任何未知参数被**静默忽略**，等同不带参数运行——会直接轮换本 profile（2026-10-18 实测：拿 `--help` 探测用法，误把 secretary 轮换了）。要看用法就读文件头注释，别拿未知参数试。
- `session.resume` 的返回是整份会话状态（本机实测约 58 万字符，含 system_prompt/工具表）；只取 `session_id`，别整包 `print` 或落日志。
- 回滚已实测可行：`session.archive {新行, archived:true}`（标题自动让出）→ `session.archive {旧行, archived:false}` → `session.title {旧行 runtime id, title:'Bot Chat'}`；核验 `session.list` 持有者与 `profiles.list.canonical_session` 都回到旧 id、DB 旧行 `archived=0/title='Bot Chat'`、新行 `archived=1/title=NULL`。

### 来自 simu-1

- Windows 工具 Python 缺 `websockets`、自动 re-exec 后返回空输出时，不把 `exit 0` 当作预检成功；直接用探测得到的 Hermes venv Python 重跑 `--dry-run`，看见确切目标后再执行。若空输出发生在正式轮换而非 dry-run，先回读 canonical/旧新行再决定下一步，禁止盲目重复轮换。

