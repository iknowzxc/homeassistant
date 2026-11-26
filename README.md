# homeassistant

Home Assistant（HAOS）在 192.168.5.249 上 `/config` 的 YAML 层版本库。

这个仓库的目标读者是"下一次接手改配置的 AI 或人"：它只存**配置本身**和**改配置必须知道的不变量**，不存任何凭据、运行态数据或设备注册表。

## 文件构成

- `configuration.yaml` — 加载 `default_config`、frontend 主题目录、include 三个 YAML，定义 `rest_command`（Server酱 ×2、WxPusher ×1）与 `input_number` helper
- `automations.yaml` — 16 条自动化（全部业务逻辑都在这里）
- `scripts.yaml` — 1 个可复用脚本 `treeow_581914_apply_fan_speed`
- `secrets.yaml.example` — `secrets.yaml` 的字段清单（真文件不入库）
- 不入库但线上存在：`secrets.yaml`、`scenes.yaml`、`themes/`、`.storage/`、`home-assistant_v2.db`（见 `.gitignore`）

## 设备

设备通过 HACS 自定义集成 `treeow`（走 `eziotes.treeow.com.cn` 云端轮询）接入，集成把每台设备展开成同一套实体后缀。

- `311806` — 次卧 G2 Pro 加湿净化一体机
- `581914` — 客厅 G2 Pro 加湿净化一体机
- `214644` — 客厅空气净化器（K3）
- `531648` — 已废弃的旧设备 ID；DB 里仍有其历史实体，配置里不要再引用

实体命名一律是 `<domain>.treeow_<设备ID>_<后缀>`，常用后缀：

- 传感器 `sensor`：`humidity`、`temp_indoor`、`pm25`、`water_table`、`pump_run`、`fan_run`、`fault`
- 开关 `switch`：`switch`（整机）、`child_lock`、`smart_mode_switch`
- 选择器 `select`：`mode_switch`、`mode`、`fan_speed`

## 设备差异（最容易写错的两处）

- **K3（214644）没有 `mode_switch`，档位实体叫 `fan_speed_enum`。** G2 Pro 的档位是 `fan_speed`。两者档位选项都形如 `1gear`…`6gear`。
- **客厅（581914）的 `temp_indoor` 上报华氏度。** 需要摄氏时用 `{{ (tv - 32) * 5 / 9 }}`，判断条件是 `tv > 50`（正常室温不可能 >50°C，所以这是安全的单位探测），不要把判断写成"有没有单位"。
- `water_table` 是十进制阶梯值（0/10/20…/100），且来自云端轮询，会出现 20 秒内 40→50→40 的抖动。任何基于它的阈值判断都必须容忍这种回踩。

## 自动化分组（16 条）

次卧 G2 Pro（7 条）
- 湿度 <50 持续 1 分钟：开加湿+净化，切 humidify，10 秒后按时间段 manual/sleep，1 档
- 湿度 >60：切 purify（PM2.5 ≥20 走净化 1 档，否则 sleep）
- PM2.5 >20：开净化，保持当前 humidify/purify
- PM2.5 <10：切 sleep
- 20:00：切 sleep（顺带开童锁、关智能模式）
- 09:00：唤醒并延时 2 分钟按湿度/PM 分流复查
- 白天按 PM2.5 四段动态调档（<20 sleep，20/40/60/80 分界，1~4 档）

客厅 G2 Pro（6 条）
- 与次卧同构，但档位统一调脚本 `treeow_581914_apply_fan_speed`，阈值只在 `scripts.yaml` 维护一处
- "空气达标切换低档"直接写 `1gear`，没有走脚本（历史遗留，可收敛）

客厅空气净化器 K3（2 条）
- PM2.5 五档映射：<20 → 2gear，20/30/40/50 分界 → 3/4/5/6 档
- 09:00 开机后延时 2 分钟复查再定档

推送（1 条）
- `加湿器水量报告推送`（id `1792800000016`）：WxPusher 主题 48733，详见下节

## 不变量 —— 改任何一条自动化前先读这段

设备唤醒序列是固定顺序，顺序和时间都不能改：

1. `switch.turn_on` 整机
2. `switch.turn_on` `child_lock`
3. `switch.turn_off` `smart_mode_switch`（不关掉，云端智能模式会覆盖你写的档位）
4. 写 `select.*_mode_switch`（`mode_humidify` 或 `mode_purify`）—— 决定"干哪件事"
5. `delay: 00:00:10` —— 等设备回读状态，**不能省**，省了后面的写入会被设备回读覆盖
6. 写 `select.*_mode`（`manual` 或 `sleep`）+ 写档位 —— 决定"怎么干"

`mode_switch` 与 `mode` 是两个维度，不是二选一；先写前者，10 秒后再写后者。

## 水量推送的去抖机制

两个 helper 记录"上次已推送的水位"，`initial: 100` 表示已武装：

- `input_number.treeow_311806_water_notified`（次卧）
- `input_number.treeow_581914_water_notified`（客厅）

规则：

- 触发：两台 `water_table` 的 `state` 变化 + `for: 1 minute`，`unknown`/`unavailable` 不触发（不再用 `numeric_state below:` —— 它没有迟滞，水位回踩阈值上方再掉下来会二次推送）
- 推送：当前水量 <50 且比上次推送值低 5 点以上，并且只认 50 / 20 两档（`n >= 50` 算跌破 50 档，`n >= 20 and w < 20` 算跌破 20 档）
- 推送后把当前水位写回 helper，所以同一水位永不重复提醒
- 回升 20 点以上（真补水）才重新武装；回踩 10 点（传感器抖动）不算
- 报告正文固定只列 `<50` 的机器，所以报告里永远不会出现 🟢（≥70 的机器进不了筛选）

## 部署流程（改完必须走完）

线上 `/config` 是部署目标，仓库才是源。不允许在网页 UI 或 SSH 里直接改 YAML —— 那样仓库立刻和线上分叉。

1. 在本仓库改文件，`git commit`
2. 对比：`md5sum /config/automations.yaml` 与本地文件，确认线上没有手工改动会被覆盖
3. 备份：`mkdir -p /config/backup-yaml-$(date +%Y%m%d-%H%M%S)` 并把要覆盖的文件复制进去（线上已有此约定）
4. 上传：scp/sftp 覆盖 `/config/*.yaml`
5. 校验：`ha core check`（约 6 秒，捕获 schema 错误，不影响运行中的系统）
6. 生效：`ha core restart`（约 45 秒）。新增顶层键（如 `input_number:`）必须整机重启，`ha core reload` 不会加载新集成
7. 验证：查 recorder 确认新实体出现、自动化为 `on`，并 `ha core logs | grep -aiE 'warn|error|invalid'` 确认重启后没有新增错误（`custom_components.treeow` 的云端超时是既有噪音）

## 排查历史与触发

recorder 库 `/config/home-assistant_v2.db` 是唯一可靠的事实来源（无需 token）：

- `states` + `states_meta` 按 `last_updated_ts` 排序，可回放任意实体的状态序列
- 带 `for: minutes: 1` 的触发器，自动化实体状态变化时间 = 传感器状态变化时间 + 60 秒，可据此把"某次推送"精确对应到"哪次跨界"
- 不要 `cat` 这个库（170 MB+），用 `sqlite3` join 查询

## 凭据

`secrets.yaml` 不入库，字段见 `secrets.yaml.example`：`klmfather_url`、`klmmother_url`（Server酱，当前没有自动化在用）、`wxpusher_app_token`（水量推送用，主题 48733 硬编码在 `configuration.yaml` 的 `rest_command.wxpusher_topic` 里）。

## 已知缺口

- `configuration.yaml` 引用的 `scenes.yaml` 与 `themes/` 不在仓库里（线上存在）——直接拿本仓库启动会报加载错误
- `README` 之外没有测试；改动只能靠 `ha core check` + recorder 回放验证
- `sensor.treeow_*_water_table` 没有 `unit_of_measurement`，所有水位比较都按裸数值处理
