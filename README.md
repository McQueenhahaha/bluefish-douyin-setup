# 大肥鱼接入抖音：我的本机方案

这份仓库记录我目前在 Windows 上实际使用的“大肥鱼接入抖音”方案。它不是把网上几种做法拼在一起，也不是官方的一键安装包；文档中的链路来自我本机正在使用的 `D:\AI\bluefish` 部署，公开时已移除账号、UID、令牌、会话 ID、Cookie、聊天记录和人格记忆。

> 当前基线（2026-10-31）：DeepSeek Harness Desktop `0.2.1-rc.5`、Node.js `24.19.0`、Chrome `156.0.8078.74`、Windows 11。

## 这套方法实现了什么

- 大肥鱼使用一个独立抖音小号，不碰主号浏览器资料。
- 主人在抖音私信小号，消息进入大肥鱼原来的 DSH 主会话。
- 大肥鱼的回复从同一条会话返回抖音，QQ 和抖音看到的是同一个人格、记忆与关系状态。
- 能接收普通文字、图片、表情包和视频分享；视频分享可先提取详情、热评、画面与音频转写，再交给模型理解。
- 回复按接近真人的速度输入；输入过程中如果主人又发来消息，会取消旧句并重新考虑。
- 遇到验证码、风控或连续发送失败会自动熔断，避免继续撞平台限制。
- 登录失效或桥接异常会由看门狗检测，并通过 QQ 或 Windows 通知提醒。

不包含的能力：自动点赞、自动评论、自动关注、批量私信、直播弹幕操作或绕过抖音风控。

## 实际架构

```text
主人 QQ ── NapCat / OneBot 11 ─┐
                                ├─ dsh-qq-bridge ─ DSH 主会话
主人抖音 ─ Chrome 私信页 ─ 抖音私信桥 ┘                  │
                                                          ├─ 人格
大肥鱼浏览抖音 ─ Playwright + 独立 Chrome Profile ────────┤  记忆
                                                          └─ 工具
```

四层各自负责：

1. **DSH 桌面端**：模型、人格、记忆、工具和主会话。
2. **dsh-qq-bridge 的本机定制版**：统一 QQ/抖音入站，按来源把回复送回正确平台。
3. **抖音私信桥**：使用 Playwright 控制独立 Chrome Profile，只盯一个许可会话；在本机 `127.0.0.1` 提供受令牌保护的接口。
4. **启动脚本、计划任务与看门狗**：让 NapCat、DSH 和抖音桥一起启动，掉线时检查和恢复。

这里最重要的设计不是“让模型操作抖音网页”，而是“把抖音变成大肥鱼现有主会话的一个入口”。QQ 和抖音不能各建一条互不相干的会话，否则人格和短期上下文会分裂。

## 准备工作

### 硬件与账号

- Windows 电脑一台，建议长期在线。
- 一个专门给大肥鱼使用的抖音小号。
- 如需 QQ 通道，再准备一个 QQ 小号。
- Chrome、Node.js 24、DSH Desktop。

不要让自动化使用自己的日常抖音主号。独立账号和独立 Chrome Profile 可以减少隐私混用，也便于单独退出和清理登录态。

### 上游组件

- [DeepSeek Harness](https://github.com/deepseek-ai/DeepSeek-Harness)：大肥鱼的运行环境。
- [dsh-qq-bridge](https://github.com/Yuuyuko-uu/dsh-qq-bridge)：DSH 与 OneBot 11 的双向桥；我的机器使用了本地定制分支。
- [NapCatQQ](https://github.com/NapNeko/NapCatQQ)：QQ 的 OneBot 11 HTTP 后端。
- [Playwright](https://playwright.dev/)：控制专用 Chrome 会话。

抖音私信部分是本机定制，没有可直接安装的官方插件。网页 DOM 和内部数据结构随时可能变化，所以应当把它视为需要维护的个人自动化，而不是稳定公共 API。

## 目录规划

我的实际目录是：

```text
D:\AI\bluefish\
├─ .dsh\                         # DSH_HOME、profile、桥接配置、记忆
├─ napcat\                       # NapCat 与 QQ 小号
├─ vendor\dsh-qq-bridge\        # 本地定制桥
├─ tools\douyin.mjs             # 只读浏览抖音
├─ tools\douyin-dm.mjs          # 抖音私信常驻桥
├─ tools\douyin-inbound.mjs     # 入站去重与系统卡过滤
├─ secrets\douyin-chrome\       # 独立 Chrome Profile，绝不上传
├─ secrets\douyin-dm.json       # 私信桥配置，绝不上传
├─ logs\douyin-dm.log           # 运行日志，绝不上传
├─ tmp\douyin-in\               # 临时下载的私信图片，绝不上传
├─ start-bluefish.ps1
├─ stop-bluefish.ps1
└─ register-bluefish-tasks.ps1
```

路径不必与我完全一致，但如果要修改根目录，必须把脚本中的绝对路径一次性改全。不要一半放在 `C:`、一半照抄 `D:` 后再逐个排错。

## 第一步：安装和配置 DSH

1. 安装 DSH Desktop，并正常完成模型配置。
2. 把 DSH 的用户目录固定到大肥鱼目录：

   ```powershell
   [Environment]::SetEnvironmentVariable('DSH_HOME', 'D:\AI\bluefish\.dsh', 'User')
   ```

3. 重启 DSH，确认使用的是 `desktop` profile。
4. 安装 QQ Bridge：

   ```text
   github:Yuuyuko-uu/dsh-qq-bridge
   ```

   可以在 DSH 的“设置 → 插件 → 安装”中粘贴；安装后重启 DSH。
5. 在“设置 → 通到 QQ”中先保持“试运行”开启，选中大肥鱼的主会话。

我的 `desktop` profile 同时加载这些包：

```text
@deepseek-ai/dsh-base
@deepseek-ai/dsh-web-app
dsh-qq-bridge
dsh-fish-memory
dsh-pet
dsh-token-optimizer
dsh-context
dsh-mcp-connector
```

抖音接入的最低要求是 DSH、一个固定主会话，以及能接收外部消息并把最终回复交给本机桥的插件。桌宠和 Token 优化器不是必需项。

## 第二步：配置 QQ 通道（可选但建议保留）

这套部署用 QQ 做日常入口、故障通知和抖音掉登录后的兜底提醒。

1. 安装 NapCat，并用大肥鱼 QQ 小号登录。
2. 开启 OneBot 11 HTTP 服务，地址限定为：

   ```text
   http://127.0.0.1:3000
   ```

3. 给 HTTP 服务设置访问令牌。
4. 在 DSH 的“通到 QQ”设置中填写地址和令牌，选择同一个主会话。
5. 先点测试发送，确认后再关闭“试运行”、开启“总开关”。

我的当前逻辑只需要本机 HTTP，不需要把 OneBot WebSocket 或管理后台暴露到局域网/公网。

## 第三步：准备抖音专用浏览器环境

项目安装：

```powershell
cd D:\AI\bluefish
npm install playwright-core
```

浏览器使用持久化目录：

```text
D:\AI\bluefish\secrets\douyin-chrome
```

关键设置：

- 使用系统已安装的 Chrome，而不是和自己的 Chrome 共用 Profile。
- `headless: false`。我实测无头模式更容易进入验证码中间页。
- 正常运行时把窗口移到屏幕外并静音；登录时再移回屏幕。
- 保留 Chrome 自己的沙箱，不使用 `--no-sandbox`。
- 语言环境设为 `zh-CN`。

第一次登录由人工完成：

```powershell
node D:\AI\bluefish\tools\douyin.mjs login
```

使用大肥鱼抖音小号扫码；如页面要求短信验证，也由本人手动完成。不要自动处理验证码，不要把 Cookie 导出到配置文件。

## 第四步：配置抖音私信桥

复制 [`examples/douyin-dm.example.json`](examples/douyin-dm.example.json) 为：

```text
D:\AI\bluefish\secrets\douyin-dm.json
```

填写四类值：

- `端口`：本机私信桥端口，我使用 `3091`。
- `令牌`：至少 32 字节的随机值，不能和 NapCat/模型 API Key 共用。
- `主人.uid`：允许进入 DSH 的抖音账号 UID。
- `主人.sec_uid` 与 `主人.昵称`：用于定位和显示允许的会话。

桥只监听 `127.0.0.1`，并要求每个请求带 `x-token`。任何带浏览器 `Origin` 的请求都拒绝，以免本机网页跨站调用。

### 为什么只允许一个会话

私信页可能同时出现陌生人、系统卡片和其他会话。我的方法不遍历全部私信，不把“新消息列表”直接交给模型，而是按配置中的 UID 找到许可会话，只读取这个会话里对方发来的消息。

这同时降低两个风险：

- 陌生人的文本不会直接变成模型指令。
- 私信列表中的其他私人内容不会进入日志、模型上下文或截图。

## 第五步：让 QQ 与抖音共用同一个 DSH 会话

QQ Bridge 的本机配置里，以下三项指向同一个主会话：

```text
私聊会话 = <同一个 DSH session>
目标会话 = <同一个 DSH session>
抖音会话 = <同一个 DSH session>
```

同时配置：

```text
抖音接口 = http://127.0.0.1:3091
抖音令牌 = <与 douyin-dm.json 相同的随机令牌>
轮询间隔毫秒 = 2000
合并窗口秒 = 5
插话就改口 = true
模拟打字 = true
```

不要把真实 session ID 写进公开仓库。它不是 API 密钥，但能暴露本机内部结构，也会让别人误以为可以照抄。

### 消息方向

入站：

```text
抖音私信页 → douyin-dm.mjs /recent → dsh-qq-bridge → DSH 主会话
```

出站：

```text
DSH 最终回复 → dsh-qq-bridge → douyin-dm.mjs /send → 抖音输入框
```

桥会记录消息来源，因此“大肥鱼在 QQ 里主动说话”和“被抖音私信叫醒”不会混到错误平台。

## 第六步：处理不同类型的入站消息

### 普通文字

从抖音消息气泡的页面数据中读取消息 ID、发送人、会话、正文和时间。只接收：

- 发送人 UID 等于许可 UID；
- 会话 ID 包含许可 UID；
- 不是自己发出的；
- 未撤回；
- 有服务器消息 ID。

### 图片与表情包

使用当前登录浏览器的请求上下文下载资源，限制：

- 单张不超过 5 MB；
- 只写入临时目录；
- 根据文件头判断 GIF/PNG/WebP/JPEG；
- 把本机路径作为图片段交给 DSH 的视觉模型；
- 不把图片提交到 GitHub。

### 视频分享

一次抖音分享可能生成“附言 + 视频卡片”两条 DOM 消息。我的入站过滤会：

1. 优先处理带附言的分享；
2. 同一视频在约 5 秒内只接收一次；
3. 跳过“获得火花见面礼”等已知系统卡；
4. 为视频提取简介、作者、统计、热评；
5. 视频取 12 个时间点拼成画面图，图文取前几张；
6. 能取得音轨时进行转写；
7. 把详情和图片一起交给模型，避免只看标题就回答。

## 第七步：安全地发送回复

我的发送逻辑有以下约束：

- 文字最多 500 字；长回复由上层桥拆段。
- 每字约 60–320 ms，并允许先停顿片刻模拟思考。
- 输入期间发现新消息就清空输入框，不发送已经过时的旧句。
- 按下回车后必须看到新的服务器消息 ID 才算发送端确认成功。
- 第一次无回执时先等待，排除迟到回执，再最多重试一次。
- 图片只接受 JPG/PNG/GIF、最大 5 MB。
- 抖音可能对图片做静默过滤，所以发送端出现气泡不等于收件端必然收到；文档和日志必须如实区分。

### 熔断规则

出现以下任一情况，暂停抖音收发 60 分钟：

- 验证码或安全验证；
- 操作频繁、异常、风险、违规、封禁或限制提示；
- 连续两条消息没有确认发送成功；
- 视频详情接口连续返回异常空结果。

熔断是为了停止自动化，不是为了绕过限制。到期后仍有验证码时，应人工检查账号状态。

## 第八步：一键启动与停止

我的桌面快捷方式调用：

```text
启动：powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "D:\AI\bluefish\start-bluefish.ps1"
停止：powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "D:\AI\bluefish\stop-bluefish.ps1"
```

启动顺序：

1. 如果 `3000` 未监听，启动 NapCat 小号。
2. 如果 DSH Desktop 未运行，启动桌面端。
3. 如果存在 `secrets\douyin-dm.json` 且 `3091` 未监听，后台启动抖音私信桥。
4. 等待 DSH Host 和 NapCat 就绪；超时则弹出明确提示。

停止脚本按路径和命令行精确识别进程，只结束本项目的 DSH、NapCat、QQ 和抖音桥，不结束自己日常使用的 QQ/Chrome。

## 第九步：开机自启、看门狗与备份

我注册了五个当前用户级计划任务：

| 任务 | 触发方式 | 用途 |
| --- | --- | --- |
| Bluefish Autostart | 登录后 30 秒 | 启动 NapCat、DSH Desktop 与抖音桥 |
| Bluefish Watchdog | 每 5 分钟 | 检查 QQ、OneBot、DSH、桥接和抖音登录 |
| Bluefish Restart | 按需触发 | 等会话空闲后安全重启 DSH |
| Bluefish Backup | 每天 04:00 | 备份人格、记忆与配置，不备份 secrets |
| Bluefish Checkup | 每周日 20:00 | 生成运行体检报告 |

备份脚本必须明确排除：

```text
secrets/
logs/
tmp/
**/*cookie*
**/*token*
**/*session*
```

仅靠 `.gitignore` 不够。公开前还要检查 Git 历史，因为“删掉文件再提交”不会从历史中移除密钥。

## 第十步：验证

按下面顺序验证，出了问题容易定位：

1. **DSH**：桌面端能打开主会话，模型可正常回复。
2. **QQ 后端**：`127.0.0.1:3000` 可用，Bridge 测试消息成功。
3. **抖音登录**：专用 Chrome Profile 打开抖音后仍是小号登录状态。
4. **私信桥**：`127.0.0.1:3091/status` 在带正确 `x-token` 时返回在线。
5. **入站**：主人从抖音发一句唯一测试文本，DSH 主会话只出现一次。
6. **出站**：大肥鱼回复抖音，日志记录服务器回执；再从收件端确认实际收到。
7. **跨平台连续性**：先在 QQ 说上半句，再在抖音接着说，确认仍是同一上下文。
8. **插话取消**：发送一条较长回复，在输入过程中再发消息，确认旧句未发出。
9. **重启恢复**：重启 DSH，不退出抖音 Chrome Profile，确认桥重新在线且没有重复发旧回复。
10. **故障提醒**：人工退出抖音小号，确认看门狗报告“未登录”，而不是误报“找不到会话”。

不要用真实隐私内容做测试；使用“bluefish-test-日期-序号”一类可识别文本。

## 常见问题

### 状态显示“未登录”

重新运行 `douyin.mjs login`，人工扫码并完成短信验证。仅有 Cookie 不代表登录流程已完成，必须确认扫码/验证码/密码登录框都消失。

### 状态显示“找不到会话”

先确认主人账号已经主动私信过小号；再核对许可 UID。不要为了找会话而遍历、打印全部私信对象。

### 发送端有气泡，但主人没收到图片

这是抖音可能出现的静默过滤。发送端回执只能证明网页账号提交了消息，不能冒充收件端验收。降低图片频率，并为每次图片发送配一条能独立成立的短文字。

### 一次视频分享触发两次回复

检查是否同时收到了附言和视频卡。应当先处理附言，并在短窗口内按视频 ID 合并，而不是单纯按正文去重。

### DSH 启动后出现重复回复

检查是否同时运行了旧 Web Host 和 Desktop Host，或加载了两份 QQ Bridge。迁移到桌面端时要停掉旧的 `dsh web` 进程。

### 更新后页面选择器失效

抖音网页版不是稳定 API。先保存脱敏截图和局部 DOM 证据，再做最小修改；不要连续盲点按钮或提高重试次数。

## 隐私与合规边界

- 只自动化自己控制的账号，只读取明确许可的私信会话。
- 不绕过验证码、风控、访问控制或平台限制。
- 不做营销群发、刷量、批量互动或账号养号。
- 不公开 Cookie、手机号、抖音号、UID、QQ 号、令牌、API Key、会话 ID、聊天记录和关系记忆。
- 日志只保留定位问题所需的最少信息，并设置轮换/清理策略。
- 抖音网页结构变化后先停桥检查，不让自动化在未知页面上继续操作。

更完整的发布检查见 [SECURITY.md](SECURITY.md)。

## 与公开项目的关系

这份仓库是对我个人部署方法的说明，不代表 DeepSeek、DSH、抖音、QQ、NapCat 或相关插件作者。文档不包含上游项目源代码；各组件的安装、许可与更新以各自仓库为准。

## License

文档与示例配置使用 [MIT License](LICENSE)。第三方软件及其商标、代码和素材仍归各自权利人所有。

