# Watch Health Inject · 手表健康注入

不知道有没有倒霉鬼和我一样小米10pro读不了睡眠数据，同时我也是个root不了的iqoo手机，所以做了这个......

把智能手环/手表（经 **Gadgetbridge**）的 **步数 / 心率 / 压力 / 卡路里 / 睡眠**数据，在每次发消息前自动注入为 **Operit 对话附件**。

> 适用场景：Gadgetbridge 已能连上你的手环，但系统（或你）拿不到新鲜数据 —— 比如 OEM 限制后台定时任务、导出频率过低、数据只在解锁时才刷新。本插件用**广播主动触发同步 + 强制导出**绕开这些限制。

---

## 它解决什么问题

很多 Android 定制系统会限制后台应用的闹钟调度，Gadgetbridge 自带的定时导出（默认每 1 小时一次）因此经常延迟。结果是：手环本地一直在记录，但数据库里的数据停在一小时前。

本插件在**每次发消息前**主动发广播触发一次同步与导出，全程 **fire-and-forget（零等待）**，所以你发下一条消息时就能拿到接近实时的数据。

---

## 工作原理

### 数据链路

```
手环 ──蓝牙(RFCOMM)──> Gadgetbridge 内部库
                          │
       插件发 ACTIVITY_SYNC 广播（主动拉取）
                          │
                          ▼
              内部库更新 ──导出广播──> Gadgetbridge.db
                          │
                          ▼
              插件读 DB ──> 注入【手表健康】附件
```

### 插件调用链（伪代码）

```
用户发消息（before_process hook）
  → triggerSyncAsync()                       // 发 ACTIVITY_SYNC，fire-and-forget
  → triggerExportIfStale(dbPath, force=true) // 强制导出，绕开 5 分钟阈值
  → buildWatchHealthContent(dbPath)          // 读 DB
  → 注入附件（数据滞后一条消息，下一条即最新）
```

> **关于"滞后一条"**：第一条消息触发同步时，本条读到的仍是上次的数据；同步在后台完成后，**下一条消息自动吃到新数据**。实测从触发到生效约 25 秒。
> 这是刻意的取舍 —— 零等待换取"消息不卡顿"，代价是首条消息数据略旧。

---

## 安装

1. 下载 Release 中的 `watch_health_inject_newXX.toolpkg`
2. 在 Operit 中导入该包
3. 在**工具箱设置**里配置 Gadgetbridge 导出的数据库路径（见下）
4. 确保 Gadgetbridge 已连接手环、服务存活

### 需要配置的路径

| 配置项 | 说明 | 示例 |
|---|---|---|
| 数据库路径 | Gadgetbridge 导出的 `.db` 文件 | `/sdcard/Download/手环/Gadgetbridge.db` |
| 睡眠 bin 目录 | 睡眠原始 bin 的读取目录（可选） | 见「数据链路」一节 |

> 中文路径经验证**无问题**，可以放心用。

---

## 触发命令（排障用）

Gadgetbridge 暴露了两个可用广播：

```bash
# 1) 主动从手环拉取数据（核心）
am broadcast -a nodomain.freeyourgadget.gadgetbridge.command.ACTIVITY_SYNC

# 2) 把内部库导出为 .db
am broadcast -a nodomain.freeyourgadget.gadgetbridge.command.TRIGGER_DATABASE_EXPORT
```

### 三个必须记住的坑

**坑 1：广播必须是「隐式」的，不能带 `-n`**

```bash
# ✅ 正确
am broadcast -a nodomain.freeyourgadget.gadgetbridge.command.ACTIVITY_SYNC

# ❌ 错误：带 -n 指定组件 → 被丢弃，毫无反应
am broadcast -n nodomain.freeyourgadget.gadgetbridge/...IntentApiReceiver -a ...ACTIVITY_SYNC
```

原因：`IntentApiReceiver` 没有在 manifest 静态注册，只在服务启动时**动态注册**。动态 receiver 只接收隐式广播（按 action 匹配），显式广播（带 component）投递不到。

**坑 2：不要带 `dataTypesHex` 参数**

```bash
# ❌ 带超大 hex → NumberFormatException → 进程被杀
am broadcast -a ...ACTIVITY_SYNC --es dataTypesHex 0xffffffff
```

原因：内部用 `Integer.parseInt(hex, 16)` 解析，`0xffffffff` 超出 int 上限。省略该参数即走默认全量同步。

**坑 3：手环必须已连接 + GB 服务存活**

动态 receiver 绑在服务生命周期上。**进程活着 ≠ 服务活着**。操作顺序：先打开 GB 主界面拉起服务 → 等手环连上（日志见 `Connected to RFCOMM socket`）→ 再发广播。

### 标准排障五连

```bash
pidof nodomain.freeyourgadget.gadgetbridge         # 进程在吗
dumpsys activity services nodomain.freeyourgadget.gadgetbridge | grep ServiceRecord  # 服务在吗
logcat -d | grep 'RFCOMM socket'                   # 手环连上了吗
logcat -c && am broadcast -a ...ACTIVITY_SYNC      # 发广播
logcat -d | grep -E 'Triggering activity sync|Mark device as busy'  # 成功了吗
```

---

## 数据链路细节（含 bin 文件）

### 原始数据落盘位置

Gadgetbridge 拉取的原始数据以 **bin 文件**落盘：

```
/sdcard/Android/data/nodomain.freeyourgadget.gadgetbridge/files/
└── <BAND_MAC>/                       ← 你的手环蓝牙 MAC
    └── rawFetchOperations/2026/ACTIVITY/
        ├── ACTIVITY_DAILY/DETAILS/   ← 每日明细
        ├── ACTIVITY_DAILY/SUMMARY/   ← 每日汇总
        ├── ACTIVITY_SLEEP/SUMMARY/   ← 睡眠汇总（*_v6.bin）
        └── ACTIVITY_MANUAL_SAMPLES/  ← 手动采样
```

文件名格式：`20260816T212800_00_00_00_v4.bin`
= `YYYYMMDDThhmmss` 采样时间 + 版本号（`v4` / `v6` 等）。

> `<BAND_MAC>` 请替换为你自己的手环 MAC。可通过 Gadgetbridge 设备页查看，或直接 `ls` 上级目录。

### 睡眠 bin 的每日同步

睡眠数据不经过数据库导出，而是由 **bin 文件**直接提供给插件。因为目录在 `Android/data/` 下权限受限，通常需要一条 shell 工作流定期把最新 bin 复制到可读位置：

```bash
SRC_DIR='/storage/emulated/0/Android/data/nodomain.freeyourgadget.gadgetbridge/files/<BAND_MAC>/rawFetchOperations/2026/ACTIVITY/ACTIVITY_SLEEP/SUMMARY'
DST='/sdcard/Download/Operit/tmp_bin/sleep_v6.bin'
LATEST=$(ls -t "$SRC_DIR"/*_v6.bin 2>/dev/null | head -1)
if [ -n "$LATEST" ]; then
  cp "$LATEST" "$DST" && echo "OK: $(basename "$LATEST") -> $DST"
else
  echo "NO_BIN: 源目录暂无 v6 bin"
fi
```

在 Operit 里可做成**定时工作流**（cron `0 7 * * *`，每天早 7 点 + 手动触发）。完整可导入模板见本仓库 [`workflows/sleep-bin-daily-sync.json`](workflows/sleep-bin-daily-sync.json)（导入前记得把 `<BAND_MAC>` 换成你自己的）。

---

## 显示项目配置

插件支持 9 个开关，自由控制附件里显示哪些内容：

| 开关 | 输出 |
|---|---|
| 步数 | `步数: N` |
| 心率 | `心率: N bpm (HH:mm:ss)` |
| 压力 | `压力: N (HH:mm:ss)` |
| 卡路里 | `卡路里: N kcal` |
| 睡眠时长 | `睡眠: N 分钟 (HH:mm → HH:mm)` |
| 睡眠细分 | `浅睡/深睡/REM` + `睡眠心率` |
| 今日汇总 | 静息/平均心率 + 心率范围 + 平均压力 |
| 生理期 | 开始日期 + 预计下次/排卵日 |
| 备注 | 自由文本备注 |

默认全部开启，标题行【手表健康】不受开关影响。

---

## 故障排查速查表

| 症状 | 原因 | 解决 |
|---|---|---|
| 广播发出但 1ms 被丢弃，无日志 | 带了 `-n` 显式广播 | 去掉 `-n`，只留 `-a` |
| 进程崩溃 + `NumberFormatException: ffffffff` | 带了超大 `dataTypesHex` | 省略该参数 |
| 广播成功但无 fetch 日志 | 设备未连接 / 服务被杀 | 打开 GB 主界面，确认 RFCOMM 已连接 |
| 数据停更 1 小时以上 | 长时间亮屏无解锁事件 → GB 不自动拉取 | 手动发 `ACTIVITY_SYNC`；本插件已内置解决 |
| 心率比实际晚 10-15 分钟 | 手环自身测量频率限制 | 正常现象，无需处理 |
| 压力/步数滞后约 2 分钟 | 符合 GB「刷新最小间隔=2 分钟」 | 正常现象，无需处理 |
| 插件发消息但 DB 没变 | 导出广播发了但 DB 未刷新 | 检查 GB 服务 + 手环连接后重试 |

---

## 版本历史

| 版本 | 核心变化 | 状态 |
|---|---|---|
| new1-new4 | 基础注入（读 DB 生成附件）+ 初步导出逻辑 | 历史 |
| new5-new6 | 尝试按需同步（5 分钟窗口跳过）| 历史（实测翻车） |
| new7 | 每次无条件同步 + 导出（稳，但慢 ~1.7s） | 历史 |
| new8/new9 | 预算限制 / 延迟回调方案 | 废弃（宿主回收执行上下文，回调从未执行） |
| new10 | 只发导出广播，零等待 | 历史 |
| new12 | 生理期配置 + 流式读取修复（防栈溢出） | 历史 |
| new13 | 备注功能 + debug.log 大小守卫 | 历史 |
| new14 | 9 项勾选显示 | 历史 |
| **new15** | **异步预同步：先 ACTIVITY_SYNC 再强制导出** | **当前** |

### 关键教训

- **手环数据是被动拉取的**：手环本地测到 ≠ GB 库里有；只有发 `ACTIVITY_SYNC` 才会更新。
- **GB 自动同步不可靠**：解锁事件触发 + 2 分钟限流，亮屏期间可能长时间不拉。
- **延迟回调在宿主里活不下来**：`Handler.postDelayed` 的延后逻辑在前置 hook 返回后会被回收 —— 所以最终方案选择 **fire-and-forget**。
- **打包必须保留完整结构**：`manifest.json` + `dist/` 前缀 + 子目录 `ui/`、`packages/`，缺一不可，否则报"格式错误无法导入"。
- **插件本身耗时极低**（导出约 124ms、查询约 40ms）；用户感知的等待绝大部分是 AI 回复生成时间。

---

## 依赖与兼容

- **Gadgetbridge**：实测 `0.93.x`，广播 action 名以你所用版本为准
- **手环**：实测 Xiaomi Smart Band 系列（设备类型 `MIBAND10PRO`）
- **系统**：Android 15（SDK 35）实测通过；其他版本如广播受限需自行验证
- **Operit**：支持 `before_process` hook 的版本

---

## 隐私说明

- 插件本体**只读**本机 Gadgetbridge 数据库与指定 bin 文件，**不上传**任何数据
- 所有路径均可在设置页自行配置
- 本仓库不含任何真实健康数据、设备 MAC 或个人路径（文档与工作流模板中的 `<BAND_MAC>` 为占位符，请替换为你自己的）

## License

MIT
