# AGENTS.md — 在这个仓库里改配置的约束

给下一次接手本仓库的 AI。先读 `README.md`，这里只放会写坏的规则。

## 源与部署

- 仓库是源，线上 `/config` 是部署目标。永远先改这里再部署，不要在网页 UI 或 SSH 里改 YAML —— 分叉后下一次部署会覆盖手工改动。
- 部署前必须 `md5sum /config/<file>` 与本地比对；不一致说明线上有仓库外的改动，先取回线上文件再动手。
- 部署顺序固定：备份到 `/config/backup-yaml-<timestamp>/` → 覆盖 → `ha core check` → `ha core restart` → 验证。校验失败就回滚备份，不要带着错误重启。
- 新增顶层配置键（`input_number:`、`template:` 等）必须整机重启；改已有自动化的内容走 `ha core check` + 重启也最稳。

## 绝不能提交的东西

- `secrets.yaml`、`.storage/`、`*.db*`、`home-assistant.log*`、`__pycache__/`。凭据只在 `secrets.yaml.example` 里以占位符出现。
- 从线上取回的 `.bak-<timestamp>` 文件、`core.entity_registry*` 等注册表快照。

## 改 YAML 时的硬规则

- 设备唤醒序列的六步顺序（开整机 → 开童锁 → 关 `smart_mode_switch` → 写 `mode_switch` → 等 10 秒 → 写 `mode` + 档位）不能重排，10 秒 `delay` 不能删。原因见 README。
- 阈值类逻辑不要用裸 `numeric_state below:`，`water_table` 之类的云端阶梯值会回踩触发重复告警；去抖方式见 README「水量推送的去抖机制」。
- 客厅 `temp_indoor` 是华氏度，转摄氏用 `> 50` 探测，不要想当然当摄氏。
- K3（`214644`）的档位实体是 `fan_speed_enum`，且没有 `mode_switch`；G2 Pro 是 `fan_speed` + `mode_switch`。
- 不要引用 `531648`（已废弃设备 ID），也不要改那些只存在于 DB 历史里的实体。
- 同一个阈值如果多处使用，抽到 `scripts.yaml`（客厅 G2 Pro 的档位脚本就是范例），不要复制到每条自动化里。

## 注释尺度

- 只在"为什么"反直觉处加注释：10 秒 delay、helper 的 100=已武装、°F 探测、为什么只认 50/20 两档。
- 不给每条 action 逐一注释，也不复述 YAML 已经写明的阈值 —— 注释和阈值会各自漂移。

## 验证方式

- 静态：`ha core check`。
- 运行时：查 `/config/home-assistant_v2.db` 的 `states`+`states_meta`（需 SSH 进 core-ssh，用 `sqlite3`），确认实体已生成、自动化 `on`、重启后无新增 error。
- 行为：把部署后的模板抽出来，用 Jinja2 配 mock `states()` 离线回放 recorder 里的真实历史，先证明改法能复现旧 bug 再证明能修掉，不要靠往用户微信推测试消息来验证。

## 提交信息

沿用 `fix(automation): …` / `feat(automation): …` / `refactor(automation): …` 前缀，正文用中文说明触发条件变化，一次提交只做一件事。
