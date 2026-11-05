---
name: sunlogin-remote-ops
description: "用向日葵（Sunlogin/AweSun）识别码 + 验证码接管一台 Windows 远程机（桌面/文件/批处理），完成部署替换、排障、日志取证。用户提到「向日葵识别码」「验证码」「远程连那台服务器」「远程帮我操作」「远程改配置」「远程发版」时必须使用。这是可执行的接管流程与排障手册，底层命令由 awesun-cli 提供。"
description_zh: "用向日葵识别码+验证码远程接管 Windows 服务器"
description_en: "Take over a remote Windows box via Sunlogin fastcode"
version: 1.0.0
display_name: "向日葵远程接管"
display_name_en: "Sunlogin remote takeover"
license: MIT
---

# 向日葵远程接管（识别码 + 验证码）

用户只给「向日葵识别码 + 验证码」，你要在**无人配合**的情况下接管那台 Windows 机器：看清屏幕、敲命令、传文件、验证结果、断开。

底层能力来自 `awesun-cli`（官方 `awesun-skill` 有完整命令表，见 <https://www.oray.com>）。本 skill 只回答一件事：**怎么把这件事一次做成**。

## 一、触发条件

满足任一即触发：

- 用户给出识别码或验证码，说「远程帮我弄一下 / 你连上去改 / 帮我远程操作」
- 故障点在另一台机器上，本机无法修复（线上服务跑在别的机器、端口被别的程序占用、需要改那台机器的配置）
- 需要在不打断对方使用的前提下，只做只读取证（看进程、看日志、看配置）
- 需要在远端执行一段批处理（替换 jar、重启服务、清理文件）

不触发：只是问命令怎么写、只要一份脚本让人自己跑（那属于交付脚本，不需要接管）。

## 二、所需输入

| 输入 | 必填 | 说明 |
|---|---|---|
| 识别码（fastcode） | 是 | 9 位左右数字。可能是设备固定识别码，也可能是「快速识别」的临时码（数分钟到数小时失效） |
| 验证码（访问密码） | 是 | 每次连接用，**只走 stdin，不作为命令行参数**（`--password` 参数不存在） |
| 目标用途 | 建议 | 只读排障 / 改配置 / 传文件 / 重启服务。用途决定要不要备份、要不要验证 |
| 目标机身份确认方式 | 建议 | 若不清楚是哪台，先 `hostname` + `ipconfig` 确认，别在错误的机器上动手 |

**凭证处理红线**：识别码与验证码只存在于当次会话变量中，不写进 SKILL.md、不写进设备清单、不写进日志或提交记录。验证码是一次性凭证，用完即弃。

## 三、执行步骤

### 0. 前置检查（10 秒，别跳过）

```bash
which awesun-cli || npm install -g @aweray/awesun-cli
awesun-cli login status          # 未登录则 awesun-cli login --url 让用户扫码
awesun-cli device ls -o json     # 顺带确认目标设备是否在线
```

`login status` 未登录时，连接会以「超时 / 无响应」的形式失败，很容易被误判成识别码错误。先排除它。

### 1. 建桌面会话

```bash
CODE=123456789          # 占位符，实际值由用户当次提供
PWD_=xxxxxx             # 占位符，用完即弃，不落盘
cd /tmp && printf '%s\n' "$PWD_" | timeout 120 awesun-cli session connect --type desktop --fastcode "$CODE" -o json
```

- `desktop` 是**唯一能用快码建立、且后续可驱动键鼠**的类型。
- `--type cmd` 用快码会直接报不支持；`--type file` 可建文件会话（用于传文件/浏览磁盘）。
- 返回里的 session id 通常是 1（再建一个是 2）。记住它，后面所有命令都带这个 id。

### 2. 第一步永远是截图确认现场

```bash
# 必须是本机绝对路径：Windows 用 C:/Users/<you>/AppData/Local/Temp/r1.png，macOS/Linux 用 /tmp/r1.png
awesun-cli session screenshot 1 --output "C:/Users/<you>/AppData/Local/Temp/r1.png"
```

然后读图：当前是什么界面？有没有登录/锁屏？命令行窗口在哪？浏览器开着什么？

**不要凭想象开始敲键盘。** 这一步能避免大量误操作。

### 3. 确认机器身份（不确定就做）

```bash
awesun-cli desktop mouse click 1 --x 0.37 --y 0.74 --button left   # 先点中命令行窗口
awesun-cli desktop paste 1 "hostname"
awesun-cli desktop key press 1 --keys enter --press down
awesun-cli desktop key press 1 --keys enter --press up
```

坐标是**归一化 0–1**（相对分辨率），不是像素。截图 → 估算位置 → 点击 → 再截图验证焦点。

### 4. 输入文本：一律用 paste，不用 type

```bash
awesun-cli desktop paste 1 "ipconfig | findstr IPv4"
```

`desktop type` 会被远端中文输入法截胡，`hostname` 变成一堆乱码/候选词。`paste` 走剪贴板，绕开输入法。

复杂命令（带引号、嵌套脚本）同样用 paste，但**优先写成本地 .bat 再传过去执行**——理由见第 6 步。

### 5. 每步之后都要截图验证

命令有没有执行、报了什么错、输出在哪一行，全靠 `screenshot` + 读图确认。不要连续盲发 5 条命令，然后猜结果。

### 6. 需要跑脚本时：本地写好 → 上传 → 远端执行

远端直接敲长命令有两类必然踩的坑：

- 从 Bash 调 `powershell -Command` 常被安全策略拦截；
- 引号/中文/反斜杠在桌面粘贴时会变形。

正确做法：本地写 `.ps1` + `.bat` 包装，传上去，只执行 bat 的绝对路径。

`.bat` 必须这样开头，否则中文输出全是乱码：

```bat
@echo off
chcp 65001 >nul
set HERE=%~dp0
powershell -NoProfile -ExecutionPolicy Bypass -File "%HERE%xxx.ps1"
```

`.ps1` 需带 UTF-8 BOM，否则 Windows PowerShell 5.1 会把中文读成乱码：

```powershell
$p='...\xxx.ps1'; $c=Get-Content -Raw -Encoding UTF8 $p
[IO.File]::WriteAllText($p,$c,(New-Object Text.UTF8Encoding $true))
```

### 7. 传文件

```bash
# 上传：--local 绝对路径；--remote 必须是「已存在的目录」；--override 才覆盖同名
awesun-cli file transfer 2 --type up --local "/path/to/new.jar" --remote "D:\app\backend" --override
awesun-cli file transfer status 2              # 轮询：pending / running / finished / failed
awesun-cli file ls 2 --path "D:\app\backend"   # 核对大小，别只信 status
```

- 40MB 级文件实测约 1 分钟，用后台任务 + 轮询，别干等。
- 传大文件时**先换远端文件名**（如 `new.jar`），不要直接覆盖正在被占用的文件。
- 同一个 `transfer_id` 会缓存上次的任务结果，出现「明明成功却报 failed」时，换个远端文件名重新发起。

### 8. 收尾：断开会话

```bash
awesun-cli session ls
awesun-cli session disconnect 1
awesun-cli session disconnect 2
```

## 四、异常与失败处理

| 现象 | 真实原因 | 处理 |
|---|---|---|
| `session connect requires either remote-id or fastcode` | 漏传参数 | 补 `--fastcode <码>` |
| `flag provided but not defined: -password` | 试图用参数传密码 | 删掉，改 `printf '%s\n' "$PWD_" \| ... connect` |
| `cmd` 会话建不起来 / 报快码不支持 | `cmd` 类型不支持 fastcode | 只用 `desktop`（或 `file`），命令行由桌面里开 |
| **识别码无效 / 已过期 / 设备离线** | 临时快码有效期短，或对端没开向日葵 | 让用户重新给码；`device ls -o json` 看目标是否 `online=true`。**不要连续重试超过 2 次**，先回报 |
| **连接超时 / 无响应** | 大概率是本机 `awesun-cli` 未登录，其次是对端网络/离线 | 先 `login status`；未登录则 `login --url` 扫码后重试一次；仍失败则判为网络侧，转「让用户本地执行」方案 |
| **验证码错误** | 密码错 | 向用户重新索取，**绝不猜测或尝试常见密码**；连续 2 次失败即停止并说明 |
| 桌面操作无反应 | 焦点不在目标窗口 / 远程锁屏 | 重新截图，点中窗口或先解锁；锁屏需要用户提供系统密码 |
| 输入变成乱码 | 远端中文输入法 | 改用 `paste` |
| 服务器访问本机 HTTP 返回 `HTTP:000` | 两端不同网段、无回程路由 | 放弃「HTTP 下发」，改 `file transfer` |
| 文件传输 `failed` 但文件其实已到 | transfer_id 缓存 | 换远端文件名重新发起，或用 `file ls` 核实实际结果 |
| 改完没生效 | 改的是副本，不是运行中的那份 | 先查 `netstat -ano \| findstr :端口` 拿 PID，再取该进程的命令行，确认**真实安装目录**和**真实读取的配置文件** |

## 五、注意事项

1. **先只读，后写入**。接管后先截图、查进程、看日志、读配置，形成判断再动手。远端盲改是所有事故的起点。
2. **改动前备份，且用改名而不是删除**（`xxx.jar.bak`、`nginx.conf.bak-20260101`）。远端没有版本控制兜底。
3. **不越界**。那台机器上可能跑着别的系统（CRM/ERP 等）。端口被占时的正确解法是**给自己换端口**，不是停掉别人的服务。
4. **改完必须验证到「用户视角」**：接口 200 只是第一步，最好在远端浏览器里真点一次，确认界面没有异常弹窗。
5. **区分「服务在跑」和「服务是对的」**：端口有监听不等于监听的是你的程序。查 PID 对应的命令行。
6. **耗时操作前后都要截图留证**，包括失败现场。用户要的是结论 + 证据，不是「应该好了」。
7. **凭证不落盘**。不把识别码/验证码写进任何文件、不 echo 到日志。
8. **断开前清理**：删掉临时诊断脚本，保留必要的备份文件，并告诉用户留了什么。
9. 用户可能在你操作期间使用这台机器。避免长按、避免关闭对方正在用的窗口（关之前先确认标题）。

## 六、可复用的配置方式

按「越往下越具体」分四层，前两层跨项目通用：

1. **环境变量注入凭证**（推荐）
   ```bash
   export SUNLOGIN_CODE=xxxxxxx SUNLOGIN_PWD=xxxxxx
   printf '%s\n' "$SUNLOGIN_PWD" | awesun-cli session connect --type desktop --fastcode "$SUNLOGIN_CODE" -o json
   ```
   会话结束即失效，不污染文件。

2. **站点清单**：把每台机器的「名称 / 用途 / 关键路径 / 端口」写在自己的 `references/hosts.local.md`（**已在 `.gitignore` 中，不会外泄**）。模板见 `references/hosts.example.md`。

3. **项目侧一键脚本**：把远端动作写成 `deploy/fix-xxx/apply.ps1` + `apply.bat`，本地留档、远端只执行。下次同类故障不用重写脚本，改参数即可。

4. **登录态常驻**：`awesun-cli login status` 正常时可长期复用，无需每次扫码；失效才 `login --url`。

## 七、标准动作清单（照抄即可）

```
[ ] login status 正常
[ ] device ls 确认目标在线
[ ] connect --type desktop --fastcode <码>（密码走 stdin）
[ ] screenshot → 看清现场
[ ] hostname + ipconfig 确认是目标机
[ ] 只读取证：端口占用 / 进程命令行 / 配置文件 / 日志尾部
[ ] 备份 → 改动 → 重启 → 本机自测
[ ] 外部验证（从自己机器打接口）+ 远端浏览器实测
[ ] 清理临时脚本，保留备份
[ ] session disconnect
[ ] 回报：病因 / 改了什么 / 证据 / 遗留事项
```
