# AI 提示词：复刻「胶东拍机」同款机场拍机 App

> 用法：把下面的内容**整段复制**给任意 AI 编程助手，只需修改「机场名 / 三字码 / 四字码」三个占位符，即可生成同款 App。
> 本文档就是本项目诞生的原始需求 + 全部实现细节的完整总结。

---

## 一、任务目标

帮我做一个**安卓 App（APK）**，用于在【机场名，例如：青岛胶东国际机场】拍飞机。
打开 App 即可看到**当天全部到港 / 离港航班**，显示**航班号、航司、机型、注册号、计划 / 预计 / 实际起降时间**，并支持**一键筛选「好货」**（宽体机、稀有机型、国产大飞机、货机、特殊涂装、收藏），最后产出可安装的签名 APK。

- 目标机场：【填写机场全名】
- 三字码（IATA）：【填写，例如 TAO】
- 四字码（ICAO）：【填写，例如 ZSQD】
- 机场坐标：【填写经纬度，例如 36.3619, 120.0882】
- 主用跑道（磁向）：【填写，例如 16/34 与 17/35；用于风向建议】

---

## 二、技术栈与成品形态

1. **安卓壳**：单个 `Activity` + `WebView`，加载 `assets/app/index.html`。
   - 不需要 AndroidX / AppCompat / Gradle（用 `javac + d8 + aapt2 + zipalign + apksigner` 命令行打包）。
   - `minSdkVersion 21`、`targetSdkVersion 34`，`INTERNET` 权限，开启 `domStorage`、`allowFileAccess`、`usesCleartextTraffic`。
2. **原生网络桥**（关键）：
   - `@JavascriptInterface` 暴露 `httpGet(url, headersJson, cbId)` / `httpPost(...)`。
   - 网络请求在后台线程执行，完成后通过 `webView.evaluateJavascript("window.__nativeCb(id, json)")` **异步回调**给 JS（不要同步阻塞，否则 UI 卡死）。
   - **自动附加 `CookieManager.getInstance().getCookie(url)` 的 Cookie**——这是通过 Flightradar24 的 Cloudflare 验证的唯一方法。
   - 自定义 UA（桌面 Chrome + 应用标识），超时 15s/25s，跟随重定向。
3. **前端**：`index.html` + `app.css` + `app.js`（ES5 兼容，避免 `?.`/`??`/模板字符串；Promise/async 可用）。
4. 全部数据请求走原生桥（规避 CORS），JS 侧封装 `nativeRequest(method, url, headers, body)` 返回 Promise。

---

## 三、数据源与接口细节（务必按此实现）

### 3.1 Flightradar24 机场时刻表（主力源）

```
GET https://api.flightradar24.com/common/v1/airport.json
  ?code=<三字码小写，如 tao>
  &plugin[]=schedule
  &plugin-setting[schedule][mode]=arrivals|departures
  &plugin-setting[schedule][timestamp]=<所选日期 00:00 本地时间 epoch 秒>
  &page=1&limit=100&fleet=
```

- **会被 Cloudflare 拦（403/302）**：必须先在 WebView 内 `iframe` 打开 `https://www.flightradar24.com/data/airport/<code>` 拿到 Cookie（自动过验证；如需人工验证给出「解锁 FR24」界面），再由原生桥带 Cookie 请求。
- 响应路径：`result.response.airport.pluginData.schedule.{arrivals|departures}.data[]`，每项含 `flight` 对象：

| 字段路径 | 含义 |
|---|---|
| `identification.number.default` / `.alternative` | 航班号 / 备用航班号 |
| `identification.callsign` | 呼号 |
| `airline.name` / `airline.short` / `airline.code.iata` / `airline.code.icao` | 航司信息 |
| `aircraft.model.code` / `aircraft.model.text` / `aircraft.registration` / `aircraft.country.name` | 机型代码 / 原文 / 注册号 / 注册国 |
| `airport.origin` / `airport.destination`：`name`、`code.iata`、`code.icao`、`position.latitude/longitude`、`position.region.city`、`timezone.name` | 机场信息 |
| `time.scheduled|estimated|real` × `departure|arrival` | 六时刻（epoch 秒） |
| `status.text` / `status.generic.status.text` | 状态原文 / 状态分类 |

- 分页：`page=1..4`，每页 100 条，返回不足 100 条即停止。

### 3.2 adsb.lol 实时周边飞机（仅「今天」）

```
GET https://api.adsb.lol/v2/point/{lat}/{lon}/{radiusKm}
```
- 返回 `ac[]`，字段：`hex`、`flight`（呼号）、`r`（注册号）、`t`（机型码）、`alt_baro`（**英尺**）、`gs`（**节**）、`baro_rate`（ft/min）、`track`、`lat`、`lon`、`dst`（距查询点 km）、`squawk`、`emergency`。
- **单位换算**：高度 ×0.3048 → 米；地速 ×1.852 → km/h；垂直速度 ×0.00508 → m/s。

### 3.3 OpenSky 实时兜底（仅「今天」，匿名可用）

```
GET https://opensky-network.org/api/states/all?lamin=..&lomin=..&lamax=..&lomax=..
```
- `states[]`：`[0]=hex, [1]=callsign, [5]=lon, [6]=lat, [7]=alt(m), [8]=on_ground, [9]=speed(m/s), [10]=track, [11]=vrate, [14]=squawk`
- 注意：机场周边可能无接收器覆盖，`states` 为 `null` 时要静默失败；`flights/arrival` 接口匿名 403 且只有前一天数据，不要依赖。

### 3.4 adsbdb 机型/航司补全（免费）

```
GET https://api.adsbdb.com/v0/aircraft/{注册号或 hex}   → response.aircraft.{icao_type, type, registration, mode_s}
GET https://api.adsbdb.com/v0/callsign/{呼号}          → response.flightroute.airline.{name, iata}
```
- 必须有本地缓存（30 天）+ 请求间隔 ≥1.3s + 每日配额（如 250 次），避免被限流。

### 3.5 Open-Meteo 天气（免费）

```
GET https://api.open-meteo.com/v1/forecast
  ?latitude=..&longitude=..
  &current=temperature_2m,apparent_temperature,relative_humidity_2m,precipitation,weather_code,cloud_cover,pressure_msl,wind_speed_10m,wind_direction_10m,wind_gusts_10m,visibility
  &daily=weather_code,temperature_2m_max,temperature_2m_min,wind_speed_10m_max,wind_direction_10m_dominant,precipitation_probability_max,sunrise,sunset,uv_index_max
  &hourly=temperature_2m,precipitation_probability,wind_speed_10m,wind_direction_10m,weather_code
  &timezone=Asia%2FShanghai&forecast_days=8
```
- WMO 天气码→中文/图标映射；按所选日期从 `daily.time` 找索引。

### 3.6 planespotters 飞机照片（免费）

```
GET https://api.planespotters.net/pub/photos/reg/{注册号大写}
```
- **必须在 User-Agent 里写联系方式**，否则 403：
  `User-Agent: TAOSpotter/2.4 (+https://example.com/contact)`
- 返回 `photos[]`：`thumbnail_large.src`（大图）、`thumbnail.src`、`link`（原图页）、`photographer`。
- 取前 10 张做**左右滑动轮播**（`overflow-x:auto; scroll-snap-type:x mandatory`），显示页码与摄影师，点图用系统浏览器打开 `link`；本地缓存 7 天。

---

## 四、功能规格（逐项）

### 4.1 日期系统
- 顶部 `◀ 10月24日 周六 [今天] ▶`，点日期打开日历面板（快捷今天/明天/后天 + 前后 7 天），左右箭头切换。
- **严格按计划日期过滤**：到港用「落地日期」、离港用「起飞日期」与所选日期比较，不相等则丢弃（实时信号无计划时刻时保留）。
- 跨天自动切换（用户手动选过日期则不自动跳）。
- 非今天时顶栏变橙色提示。

### 4.2 航班列表（到港 / 离港）
- 状态分组（每组可折叠，箭头 + 收起/展开文字，整行大按钮）：
  - 到港：`✈ 正在飞向青岛` → `🛬 即将落地` → `⏰ 时刻已过（状态未更新）` → `🅿 地面` → `✅ 已落地` → `❌ 已取消` → `📡 其他实时信号`
  - 离港：`🛫 即将起飞` → `⏰ 时刻已过` → `🅿 地面` → `✅ 已起飞离场` → `❌ 已取消` → `📡 其他实时信号`
- 排序：未到点按时间升序（越近越上）；已过点按时间倒序；无时刻放最后。
- 卡片（两段式，简洁）：
  ```
  12:15  [预计]  [落地]        [宽体]
  杭州 → 青岛
  MU5512 · 中国东方航空
  [波音737-800] [B-30CZ] [还有25分] [晚10分]
  ```
  - 时间大字 + 时刻类型（计划/预计/实际，蓝/橙/绿标签）+ **落地（绿）/起飞（橙）彩色标签**
  - 中文城市航线、航班号 + 航司中文名
  - 底部信息胶囊：机型全称、注册号、倒计时、延误、实时高度/距场
  - 右上角好货徽章、特殊状态徽章

### 4.3 好货筛选
- 好货专页分三组：**即将到来**（时间越近越上）→ **正在飞行** → **已落地/已起飞**（最近发生的在前）。
- 筛选 chips：全部 / 宽体 / 稀有 / 国产 / 货机 / 涂装 / 收藏。
- 顶部统计：「好货 N 架 · 宽体 x · 稀有 x · …」。
- 长按卡片：收藏此机 / 标记特殊涂装（输入名称，按注册号存本地）/ 复制航班信息。

### 4.4 特殊状态「动态」栏
- 识别：`7500/7600/7700` 紧急代码（红色闪烁）、ADS-B `emergency !== none`、备降（diverted/redirected）、取消、延误 ≥15 分、盘旋等待（启发式：中低空 1200-7500m + 距场 3-70km + 地速 250-750km/h + 垂直速度平稳 + 连续两次采样高度/距离稳定）。
- 分两组：`⚠️ 进行中 / 即将发生`（紧急优先，同级按时间近）→ `✅ 已结束`（按时间倒序）。

### 4.5 详情页
1. 航班号大字 + 状态徽章 + 航司 + 状态通俗解释（附 FR24 状态原文）
2. 飞机照片轮播
3. 时刻表网格：起飞/落地 × 计划/预计/实际（每格带日期）+ 计划飞行时长 + 倒计时 + 延误
4. 特殊状态（含 Squawk 代码解释）
5. 机场卡：中文城市 + 三字码 + 四字码 + 国家 + 时区 + 经纬度 + 机场全名
6. 飞机：航班号（含备用号）、航司 + 代码 + 简称、中文机型 + 代码 + FR24 原文、注册号、注册国、呼号、ICAO 地址
7. 实时位置：高度 / 地速 / 航向 / 垂直速度 / 距机场
8. 好货解读（每个标签一句人话解释）、拍摄建议（按机型：宽体拍抬头、国产拍特写、货机夜间注意 ISO…）、数据来源

### 4.6 天气
- 天气卡：天气图标（发光）+ 温度渐变大字 + 状况/体感/最高最低温 + 迷你风向罗盘 + 风速/能见度/降水/拍机指数四格 + 日出日落湿度 + 跑道建议 + 「点此查看预报」。
- 点击展开：大罗盘（跑道 16/34、17/35 方位线 + 风向箭头 + 建议起降端）、逐小时（12 小时）、未来 7 天播报。
- 跑道建议：按风向计算逆风分量，给出「建议 16/17 端」或「34/35 端」。

### 4.7 其他
- 每 3 分钟自动刷新 + 下拉刷新 + 手动刷新（SVG 图标绕中心旋转）。
- 打开 App：① 大字欢迎层（AI 工具说明 / 反馈邮箱 / 源码 / 复刻提示词入口）② VPN 提示（12 秒消失）③ 12 秒无数据再弹重试提示。
- 设置页：数据源开关、涂装库管理、收藏管理、App 提示词（可复制）、清理缓存、关于。
- 深色航空主题（深蓝底 #0A1622、青色强调 #35C4F0），状态色：计划蓝/预计橙/实际绿/取消红。

---

## 五、好货判定集合（照抄即可）

```
宽体: A306 A30B A310 A332 A333 A338 A339 A342 A343 A345 A346 A359 A35K A388
      B742 B743 B744 B748 B74D B762 B763 B764 B772 B773 B77L B77W B788 B789 B78X B778 B779
      MD11 DC10 IL96
      (IATA) 330 332 333 33F 33L 340 343 345 346 350 359 35K 388 744 748 74F 74N 74H 74W 762 763 764 76F 772 773 77W 77L 77F 787 788 789
稀有: C919 AJ27 ARJ21 MA60 MA600 MA700 YN2 Y12 Y8 AN12 AN26 AN124 AN225 IL76 IL62 TU204 TU154
      B703~B736 B752 B753 MD82 MD83 MD87 MD88 MD90 DC9 DC3 CRJ1 CRJ2 CRJ7 CRJ9 CRJX E135 E145 E170 E190 E195
      SU9 AT43 AT45 AT72 AT76 DH8A~DH8D BA11 B461 B462 F100 F70 SH33 SH36 SF34 BE99 C130
      (IATA) CR2 CR7 CR9 CRK E90 E75 E95 E70 E45 ER4 AT7 AT5 DH4 MA6 AR1 C91
国产: C919 AJ27 ARJ21 MA60 MA600 MA700 YN2 Y12 AR1 C91 MA6
货机(按 ICAO 呼号前缀): CKK CSS HYT CYZ HLF LHA YZR CAO CTJ CCO CJX GTI CKS CLX UPS FDX BOX AZQ ABW PAC NCA AHK GEC CMB SIL VDA RUN ADR
涂装: 按注册号匹配本地涂装库（内置 B-919A 全球首架C919、B-5425 国航冬奥冰雪号，可增删）
收藏: 用户长按收藏的注册号
```

机型中文名映射至少要覆盖：A319/A320/A321 + neo、A330-200/300/800/900、A340 系列、A350-900/1000、A380、B737-300/400/500/600/700/800/900、B737 MAX 7/8/9、B747-400/8/货机、B757、B767-200/300/400/300F、B777-200/300/300ER/200LR/777F、B787-8/9/10、C919、ARJ21、新舟 60/600、运-12、MD-11/82/83/90、CRJ-200/700/900、E-170/175/190/195、SSJ100、伊尔-76/96、安-12/26/124、图-204……

---

## 六、机场中文名映射

- **三字码 → 中文城市**（约 140 条）：HGH 杭州、HRB 哈尔滨、SHA/PVG 上海、HSN 舟山、TAO 青岛、PEK/PKX 北京、CAN 广州、SZX 深圳、CTU/TFU 成都、CKG 重庆、WUH 武汉、XIY 西安、NKG 南京、TSN 天津、CGO 郑州、CSX 长沙、KMG 昆明、SYX 三亚、HAK 海口、XMN 厦门、FOC 福州、TNA 济南、YNT 烟台、DLC 大连、SHE 沈阳、CGQ 长春、HET 呼和浩特、TYN 太原、SJW 石家庄、LHW 兰州、XNN 西宁、INC 银川、URC 乌鲁木齐、LXA 拉萨、KWE 贵阳、NNG 南宁、WNZ 温州、NGB 宁波、HYN 台州、YIW 义乌、JJN 泉州、SWA 汕头、ZUH 珠海、BHY 北海、KWL 桂林、LZH 柳州、NRT/HND 东京、KIX 大阪、ICN/GMP 首尔、PUS 釜山、CJU 济州、TPE/TSA 台北、KHH 高雄、HKG 香港、MFM 澳门、SIN 新加坡、BKK/DMK 曼谷、KUL 吉隆坡、MNL 马尼拉、HAN 河内、SGN 胡志明市、DXB 迪拜、IST 伊斯坦布尔……
- **拼音/英文城市 → 中文**（约 180 条）：Hangzhou 杭州、Harbin 哈尔滨、Shanghai 上海、Zhoushan 舟山、Wenzhou 温州、Ningbo 宁波、Qingdao 青岛、Beijing 北京、Guangzhou 广州、Shenzhen 深圳、Chengdu 成都、Chongqing 重庆、Tokyo 东京、Seoul 首尔、Los Angeles 洛杉矶、San Francisco 旧金山、Frankfurt 法兰克福……
- 显示优先级：三字码映射 → 拼音映射 → 原文。

---

## 七、UI / UX 规范

- 深色航空主题：背景 `#0A1622`，卡片 `#122638`，描边 `#1E3A57`，主强调 `#35C4F0`，次强调 `#19E3B1`。
- 状态色：计划蓝 `#3D8BFF`、预计橙 `#FFB03A`、实际绿 `#2ED47A`、取消红 `#FF5C5C`、宽体橙、稀有紫 `#B07CFF`、货机青 `#39E0D4`、涂装粉 `#FF6EC7`、收藏黄 `#FFD75E`。
- 顶栏：logo 42px + 应用名 + 副标题；日期栏 40px 高按钮；标签页 4 个（到港/离港/好货/动态）可横向适配。
- 卡片：圆角 16px、内边距 13px、好货左侧渐变色条。
- 分组头：48px 高整行按钮、圆角 14px、右侧「收起/展开」。
- 天气卡：温度渐变文字（`-webkit-background-clip:text`）、图标发光、指标四格。
- 所有按钮有点击缩放反馈；弹层底部圆角 20px。
- 中文字体优先 `PingFang SC / HarmonyOS Sans / MiSans`。

---

## 八、打包与踩坑（重要）

### 无 Gradle 打包流程（Windows）
```powershell
javac -encoding UTF-8 -source 1.8 -target 1.8 -bootclasspath android.jar -d classes src\...\MainActivity.java
java -cp build-tools\lib\d8.jar com.android.tools.r8.D8 --release --lib android.jar --min-api 21 --output dex classes\*.class
aapt2 compile --dir res -o res.zip
aapt2 link -o base.apk -I android.jar --manifest AndroidManifest.xml res.zip
# assets 与 classes.dex 用 .NET ZipArchive 以「正斜杠条目名」写入（见坑 1）
zipalign -f -p 4 base.apk aligned.apk
keytool -genkeypair -keystore app.keystore -alias app -keyalg RSA -keysize 2048 ...
apksigner sign --ks app.keystore --ks-pass pass:*** --out app.apk aligned.apk
```

### 必须知道的坑
1. **aapt2 打 assets 会写反斜杠条目名**（`assets\app\index.html`）→ Android 找不到 `assets/app/index.html` → 白屏 `ERR_FILE_NOT_FOUND`。解决：**不要用 `aapt2 -A`**，用 .NET `System.IO.Compression.ZipArchive` 以显式 `assets/app/index.html` 正斜杠路径写入。
2. **JS 对象键**：以数字开头（`8L:`、`73F:`、`9C:`）或含撇号/空格（`Xi'an:`、`Hong Kong:`）**必须加引号**，否则整个 JS 语法失败 → 白屏。打包前跑 `node --check app.js`。
3. **PowerShell 5.1 运行含中文注释的 .ps1 需要 UTF-8 BOM**，否则中文被按 GBK 误读导致语法错误。
4. PowerShell 5.1 原生命令的 stderr 会触发 `NativeCommandError` → 用 `2>&1 | ForEach-Object { Write-Host $_ }` 消费并检查 `$LASTEXITCODE`。
5. `aapt2/zipalign` 对**中文路径**支持差 → 把源码复制到 `%TEMP%\taospot_build` 这类 ASCII 目录再构建，最后把 APK 复制回交付目录。
6. WebView 页面加载失败要 `onReceivedError` 兜底（`loadDataWithBaseURL` 注入友好 HTML + 重新加载按钮），别让用户看到系统裸报错页。
7. 日期过滤要**按计划时刻的本地日期**过滤（到港=落地日、离港=起飞日），否则 FR24 的跨日窗口会把次日航班混进来。
8. 刷新图标用 **SVG 图标**（旋转绕几何中心），字符 `⟳` 会因字形偏置旋转发歪。
9. 应用图标用字符 ✈ 渲染时，**要先渲染测量白色像素包围盒再平移**做像素级居中。

---

## 九、测试要求

1. `node --check assets/app/app.js`（语法）。
2. 用 Node `vm` + 最小 DOM 桩加载 `app.js`，对核心函数做单元测试（≥50 项）：
   - 日期：`todayStr/shiftDate/dayStartEpoch/relDayTag/dateLabel`
   - 转换：`csToNo`、`cityCn/cityDisplay`、`fmtHm/fmtMonthDay/fmtCountdown`
   - 判定：`classify`（宽体/稀有/国产/货机/涂装）、`statusInfo`、`specialFlags`、`detectHolding`
   - 分组：`groupFlights`（到港/离港/好货/动态/时刻已过）
   - 排序：落地时间近的在前、已结束倒序
   - 过滤：`filterByDate` 次日照物不混入
   - 渲染：`renderAll`、`cardHtml` 不抛异常且含关键信息
   - 解析：`parseFr24Item` 完整字段
   - 天气：`weatherCodeInfo/dirName/windArrowIcon/windCompassSvg/buildWeatherSheetHtml`
3. 真实接口链路测试（可选）：Node 环境直连 adsb.lol / Open-Meteo / planespotters，验证解析与失败兜底。
4. 构建后验证 APK：`aapt2 dump badging`（包名/版本/权限）、zip 条目全为正斜杠、`apksigner verify`、图标像素居中。

---

## 十、交付物

1. 可安装的签名 APK（含自定义图标、应用名）。
2. 完整源码目录（`src/`、`res/`、`assets/`、`AndroidManifest.xml`、`build.ps1`、`gen-icon.ps1`）。
3. `README.md`（功能说明、安装、数据源、免责声明）。
4. 单元测试文件 `test-core.js` 与真实接口测试 `test-live.js`。
5. 本提示词文件（供其他机场复刻）。

---

## 十一、署名与声明要求

- 应用内每次打开显示欢迎层：
  「AI 做出来的一个小工具 · 有问题请反馈至 Lan_X86@outlook.com · 感谢支持 · 源码已在 GitHub 展示 · 如需复刻其他机场，提示词已在设置里公开」
- 设置页公开本提示词，并提供「复制」按钮。
- README 写明数据来源与免责声明（数据来自公开接口、以实际运行为准、照片版权归 planespotters 摄影师、仅供个人爱好使用）。
