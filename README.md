# 汽水音乐自动化脚本（iOS / Android）

基于 AScript 框架的移动端自动化脚本，支持 iOS 与 Android，用于自动观看汽水音乐广告以获取免费听歌时长。

## 平台切换

1. 修改 `__init__.py` 顶部的 `PLATFORM` 常量：
   - `"auto"`（默认）：自动识别，能导入 `ascript.android` 即为 Android，否则为 iOS
   - `"ios"` / `"android"`：强制指定
2. 修改 `.vscode/settings.json` 中的 `"ascript.platform"`（`"iOS"` / `"Android"`）和 `"ascript.deviceId"`（设备 IP:端口）

| 差异项       | iOS                         | Android                                   |
| ------------ | --------------------------- | ----------------------------------------- |
| App 标识     | `APP_BUNDLE_ID`（`com.soda.music`） | `APP_PACKAGE_ANDROID`（`com.luna.music`，如不对请用 AScript「开发者-查包工具」确认） |
| OCR 引擎     | `vision`                    | `paddle`（可改 `mlkit`）                  |
| 关闭续费弹窗 | 点击弹窗外遮罩              | 返回键                                    |
| 直播间兜底   | Selector 找「关闭」+ 固定坐标 | 返回键                                    |
| 找图         | 单尺度                      | 多尺度匹配（`scale_range=[0.6, 1.6]`），兼容 iOS 截取的模板 |
| 运行权限     | WebDriverAgent              | 无障碍 / Root / HID 任一可点击的模式      |

> OCR 区域按 iPhone 实测百分比设定，Android 机型若识别不到按钮，需按实际界面微调各 `rect_*` 百分比。

## 功能特性

- 自动启动汽水音乐 App
- 自动跳过开屏广告（paddleocr 精准识别）
- 支持两种广告类型：视频广告 + 直播间广告
- 广告结束后自动领取奖励（两步：点「领取成功」→ 点「领取奖励」）
- 无限循环领取模式，直到当日额度用完
- 全程 OCR + 图像识别，无需依赖控件树

## 技术栈

| 模块                            | 用途                                  |
| ------------------------------- | ------------------------------------- |
| `ascript.ios.system`            | 应用启动 / 屏幕尺寸 / 资源路径 / 通知 |
| `ascript.ios.action`            | 点击等动作执行                        |
| `ascript.ios.screen.Ocr`        | 文字识别（paddleocr / vision 双引擎） |
| `ascript.ios.screen.FindImages` | 图像模板匹配（直播间关闭按钮等）      |

## 运行环境

- **平台**：iOS（需 WebDriverAgent 连接）
- **AScript 版本**：支持 `ascript.ios.*` 命名空间的版本
- **OCR 引擎**：paddleocr（推荐，识别率高）或 vision（iOS16+ 原生）
- **HID 点击通道**：建议配置 ESP32 BLE HID 或其他外接 HID 方案，避免 WDA click 被 App 检测拦截

## 项目结构

```
qishuiAScriptProject/
├── __init__.py          # 脚本主入口（AScript 工程约定，必须叫这个名字）
├── res/
│   └── img/
│       ├── logo.png
│       ├── qishuiyinyue.png           # 汽水音乐桌面图标（兜底启动用）
│       ├── img_zhibojian-close01.png  # 直播间关闭按钮模板图
│       └── img_1785263403257.png
├── .vscode/
│   └── settings.json       # 项目配置（平台 / 设备地址 / Python 路径）
├── .trae/
│   └── rules/
│       └── ascript.md      # AScript AI 编码规则
└── .gitignore
```

## 快速开始

### 1. 环境检查

确保 AScript App 已安装在 iOS 设备上，且 WebDriverAgent 已启动并可访问。

```bash
# 查看设备状态（AScript MCP 工具）
get_device_status()
```

### 2. 设备连接

```bash
# 自动连接（推荐）
auto_connect()

# 或手动连接（扫描后用 IP）
scan_devices()
connect_device(host="192.168.1.2", port=9096)
```

> 本项目 `.vscode/settings.json` 默认设备地址：`192.168.1.2:9096`

### 3. 运行脚本

```bash
# 方式一：直接运行工程
run_project(project="qishuiAScriptProject")

# 方式二：先上传再运行
upload_file(...)
run_project(...)
```

## 脚本流程

```
启动 App
    ↓
[用户手动进入第一个广告] 或 [脚本自动找入口]
    ↓
┌───────────────────────────────────────┐
│         核心广告领取循环              │
│                                       │
│  等待广告倒计时结束（最长 30s）        │
│    ├─ 视频广告：等「领取成功」出现     │
│    └─ 直播间广告：关直播间 → 等领取   │
│          ↓                            │
│  步骤 1：点右上角「领取成功」          │
│          ↓                            │
│  步骤 2：点弹窗「领取奖励」→ 进下一轮 │
│          ↓                            │
│  成功次数 +1，重复循环                │
│                                       │
│  连续 10 轮无广告 → 自动退出          │
└───────────────────────────────────────┘
```

## 关键配置

修改 [__init__.py](./__init__.py) 头部常量即可：

| 常量             | 默认值             | 说明                                           |
| ---------------- | ------------------ | ---------------------------------------------- |
| `PLATFORM`       | `"auto"`           | 运行平台：`"auto"` / `"ios"` / `"android"`     |
| `APP_NAME`       | `"汽水音乐"`       | App 名称（兜底用）                             |
| `APP_BUNDLE_ID`  | `"com.soda.music"` | iOS Bundle ID（优先使用，更稳定）              |
| `APP_PACKAGE_ANDROID` | `"com.luna.music"` | Android 包名                             |
| `MAX_LOOP_COUNT` | `50`               | 最大循环次数（防死循环，实际核心循环另有限制） |
| `LOOP_INTERVAL`  | `3`                | 主循环间隔秒数                                 |
| `OCR_ENGINE`     | iOS `"vision"` / Android `"paddle"` | OCR 引擎，按平台自动选择 |

核心循环参数：

| 常量              | 默认值 | 说明                                               |
| ----------------- | ------ | -------------------------------------------------- |
| `MAX_IDLE_ROUNDS` | `10`   | 连续多少轮没领到奖励后自动退出（判断当日额度用完） |

## OCR 区域设计原则

所有搜索区域都**基于屏幕百分比动态计算**，适配不同分辨率（iPhone SE ~ iPhone 15 Pro Max）。

| 区域             | 用途                           | 坐标范围近似         |
| ---------------- | ------------------------------ | -------------------- |
| 开屏跳过按钮     | 跳过广告                       | x:80%~97%, y:6%~10%  |
| 中间「继续观看」 | 广告中途恢复（防误点底部下载） | x:7%~93%, y:25%~68%  |
| 弹窗「领取奖励」 | 奖励按钮（仅限中间弹窗区）     | x:13%~87%, y:25%~70% |

## 注意事项

1. **点击通道**：iOS 的 WDA click 容易被 App 检测拦截，建议配置 ESP32 BLE HID（插件 id=103）或其他外接 HID 方案。
2. **不要写 `if __name__ == "__main__"`**：AScript 工程入口不等于 `__main__`，主体代码直接写在顶层即可，[入口约定参考](./.trae/rules/ascript.md)。
3. **资源文件**：`res/img/` 下的图标和模板图需要根据实际设备 UI 重新裁剪，分辨率不匹配会降低识别率。
4. **当日额度**：汽水音乐每日有广告领取上限，连续 10 轮没领到会自动退出，不要反复空跑。

## 相关文档

- [AScript 编码规则](./.trae/rules/ascript.md)
- 主脚本：[__init__.py](./__init__.py)
