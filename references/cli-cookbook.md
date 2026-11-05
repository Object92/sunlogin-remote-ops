# awesun-cli 命令速查（实测可用）

安装：`npm install -g @aweray/awesun-cli`
若 `which awesun-cli` 为空，检查 npm 全局 bin 是否在 PATH 中。

## 登录与设备

```bash
awesun-cli login status              # 判断是否要重新扫码（连接超时先查这个）
awesun-cli login --url               # 输出登录 URL，让用户自己扫码
awesun-cli login --user <username>   # 账号密码登录
awesun-cli device ls -o json         # 字段：remote_id / name / online / os_name / cpu
```

## 建会话

```bash
# 桌面（唯一支持快码且可驱动键鼠）
printf '%s\n' "$PWD_" | timeout 120 awesun-cli session connect --type desktop --fastcode "$CODE" -o json

# 文件（浏览磁盘、传文件）
printf '%s\n' "$PWD_" | timeout 90  awesun-cli session connect --type file   --fastcode "$CODE" -o json

awesun-cli session ls
awesun-cli session status <sid>
awesun-cli session disconnect <sid>
```

> `--type cmd` **不支持快码**，会报错。要命令行就在桌面里 `win+r` → cmd。
> `--password` 这个参数**不存在**，密码只能走 stdin。

## 截图（每步必做）

```bash
awesun-cli session screenshot 1 --output "C:/Users/<you>/AppData/Local/Temp/r.png"
```
输出必须是绝对路径，正斜杠即可。截完立刻读图，别靠猜。

## 键鼠

```bash
# 坐标是归一化 0–1
awesun-cli desktop mouse click 1 --x 0.37 --y 0.74 --button left

# 粘贴（推荐，绕开输入法）；type 会被中文输入法污染
awesun-cli desktop paste 1 "hostname"
awesun-cli desktop type  1 "hostname"      # 不推荐

# 单键
awesun-cli desktop key press 1 --keys enter --press down
awesun-cli desktop key press 1 --keys enter --press up

# 组合键
awesun-cli desktop key combo 1 --keys win,r
awesun-cli desktop key combo 1 --keys ctrl,f5
awesun-cli desktop key combo 1 --keys alt,f4
```

`key press` 要成对发 down/up，只发一半有时不生效。

## 文件

```bash
awesun-cli file ls 2 --path "D:\\dir"          # JSON 里反斜杠要双写
awesun-cli file mkdir 2 --path "D:\\dir"
awesun-cli file rm 2 --path "D:\\dir\\x.bat"
awesun-cli file mv 2 --from "D:\\a" --to "D:\\b"
```

### 传输（三个必守规则）

```bash
awesun-cli file transfer 2 --type up --local "C:\\path\\new.jar" --remote "D:\\app\\backend" --override
awesun-cli file transfer 2 --type down --local "C:\\path\\" --remote "D:\\app\\backend\\app.pid"
awesun-cli file transfer status 2
```

1. `--local` 必须是**绝对路径**（写成相对路径会报「本地文件不存在」）。
2. `--remote` 上传时必须是**已存在的目录**，不是目标文件名。
3. 同名覆盖要显式 `--override`；同一 `transfer_id` 会复用上次结果，异常时**换远端文件名**重发。

大文件用后台任务 + 轮询：

```bash
for i in $(seq 1 20); do S=$(awesun-cli file transfer status 2 2>&1 | grep <tid> | awk '{print $2}'); \
  echo "$(date +%H:%M:%S) -> ${S:-?}"; [ "$S" = "finished" ] || [ "$S" = "failed" ] && break; sleep 8; done
```

## 远端执行批处理的固定套路

1. 本地写 `x.ps1`（UTF-8 **带 BOM**）+ `x.bat`（首行 `chcp 65001`）。
2. `file transfer up` 到目标目录（目录必须存在）。
3. 桌面 `win+r` → `cmd`，`paste` 绝对路径 → 回车。**不要**在 Linux/macOS 的 shell 里直接调 `powershell -Command`，会被安全策略拦。
4. 等 10–60 秒（视脚本内容），截图读输出。

## 会话资源

- 一个连接占一个 session id（1、2、3…），`file` 与 `desktop` 各占一个。
- 结束时都断掉，避免残留占用：
  `awesun-cli session ls` → `session disconnect 1`、`session disconnect 2`
