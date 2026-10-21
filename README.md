# Motion Recorder

一个独立、手机优先、无依赖的 DeviceMotion 六轴数据采集网站。它不会上传或伪造数据；真实数据只能由你在自己的手机浏览器中授权并录制。

## 文件

- `index.html`：完整网站，CSS 与 JavaScript 已内嵌。
- `CODEX_PROMPTS.md`：拿到真实 JSON 后，在现有 VS Code 工程中分阶段执行的提示词。

## 最快部署方式

手机传感器通常只在 **HTTPS** 页面或 `localhost` 中开放。电脑局域网的普通 `http://192.168...` 往往不行。

### 方案 A：GitHub Pages

1. 新建一个 GitHub 仓库，把 `index.html` 放在仓库根目录。
2. 在仓库 Settings → Pages 中选择从主分支部署。
3. 等待生成 `https://<用户名>.github.io/<仓库名>/`，用手机打开。

### 方案 B：静态托管拖放部署

把包含 `index.html` 的文件夹拖到 Netlify Drop 或其他支持 HTTPS 的静态托管服务。部署后用手机访问生成的 HTTPS 地址。

### VS Code 本地预览

VS Code 的 Live Server 可用于电脑上检查布局，但从手机访问电脑局域网地址时通常不是安全上下文，因此可能收不到传感器数据。正式采集请使用 HTTPS 部署地址。

## 手机授权

- **iPhone / iPad**：建议 Safari。打开页面后必须由你点击“启用传感器”，系统才会弹出“运动与方向访问”授权。拒绝后，可在 Safari 的网站设置或系统设置中重新允许，再刷新页面。
- **Android**：建议 Chrome。多数设备不弹单独的 DeviceMotion 对话框，但仍需通过 HTTPS 打开并点击启用按钮。
- 如果页面显示“API 不支持”或始终为 0 个样本，先换用系统浏览器，确认不是内嵌网页，再检查网站权限与 HTTPS。

## 推荐采集流程

1. 先决定固定持有/佩戴位置：脚步声可固定在鞋面或脚踝；魔法杖与枪效保持相同的握持方向。固定牢靠，避免手机跌落。
2. 选择动作、正样本，准备 3 秒，每段 1.0–2.0 秒，批量 5 次。
3. 每个动作先采 10–20 个正样本。动作要自然，强弱和快慢有少量变化，但不要在同一段里连续做很多次。
4. 再录负样本：静止、走动时的日常晃动，以及“很像但不该触发”的动作。负样本也应覆盖三种动作各自的误触场景。
5. 看最近一段的质量提示。出现“没有样本”“六轴缺失比例较高”“时间戳不递增”或实际时长明显不足时，建议删除并重录。低采样率警告不一定代表不可用，但需要在后续算法中按真实频率处理。
6. 每完成一组就导出 JSON。刷新或关闭页面会清空尚未下载的数据。

建议至少分两轮采集：第一轮用于建立规则/模板，第二轮在不同时间和环境中录制，用于验证是否会误触。不要把连续录制中相邻的样本随意拆到训练集和验证集，否则结果会显得虚高。

## 导出数据结构

根对象：

```text
schemaVersion: "motion-recorder.dataset.v1"
exportedAt: ISO 8601
dataset:
  id, createdAt, recordingCount, labels
capture:
  sensor, primaryAxes, additionalAxes, timestampFields,
  permissionState, defaultConfig
exportDevice:
  userAgent, platform, language, timeZone, screen
recordings[]:
  id
  label: { action, polarity }
  sequence: { index, total }
  startedAt, endedAt, targetDurationMs, permissionState
  device
  sampling:
    sampleCount, effectiveHz, meanIntervalMs, intervalJitterMs,
    missingAxisRatio, monotonic, actualDurationMs, quality, issues
  samples[]:
    capturedAtUnixMs     # 由事件时间戳换算的绝对 Unix 毫秒
    receivedAtUnixMs     # JavaScript 收到事件的系统时间
    relativeMs           # 相对本段开始的毫秒数
    eventIntervalMs      # 浏览器报告值，可能为 null
    acceleration: { x, y, z }
    accelerationIncludingGravity: { x, y, z }
    rotationRate: { alpha, beta, gamma }
    screenOrientationDegrees
```

原始轴缺失时写入 `null`，不会用 0 补齐。后续工程应按每段实测时间戳重采样，而不是假设固定频率。

## 质量检查的含义

- **good / 良好**：基础完整性检查通过。
- **warning / 留意**：常见情况是实测频率低于约 30 Hz，或采样间隔抖动较大。数据仍会保留。
- **bad / 需重录**：无样本、六轴缺失较多、时间戳异常或采集时长明显不足。

这些只是采集完整性检查，不代表动作分类效果。动作阈值、DTW 距离与防误触参数应在导入你的真实数据后再确定。

## 隐私与限制

- 页面没有网络上传代码，数据只在当前标签页内存中，点击导出时生成本地下载。
- 浏览器不会提供统一的手机型号字段；`userAgent` 等信息仅用于定位设备差异。
- DeviceMotion 采样率受机型、浏览器、省电状态和页面前后台状态影响。录制时保持页面在前台。
- 这个采集器不做动作识别，也不承诺 10–20 次样本足够泛化；它的任务是可靠保存原始数据和元数据。

## 下一步

把导出的一个或多个 JSON 放进你的现有工程，再从 `CODEX_PROMPTS.md` 的“阶段 0”开始逐段交给 Codex。阶段 0 只检查现有结构，不改代码；这样可以先确认哪些 UI、声音地图和 Tone.js 组件能够复用。

