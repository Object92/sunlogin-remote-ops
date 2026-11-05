# sunlogin-remote-ops

> 一个给 AI Agent 用的 skill：用户只给「向日葵识别码 + 验证码」，就能接管一台 Windows 远程机器，完成排障、改配置、换文件、重启服务、验证结果。

这不是向日葵的使用教程，也不是 `awesun-cli` 的命令手册。它解决的是**拿到两个码之后怎么把事一次做成**——包括那些只有踩过才知道的操作顺序和失败分支。

## 它解决什么问题

AI 直接用 `awesun-cli` 接管远程机，通常会在这些地方翻车：

| 坑 | 后果 | 本 skill 的处理 |
|---|---|---|
| 以为有 `--password` 参数 | 直接报 `flag provided but not defined` | 验证码一律 `printf '%s\n' \| session connect` 走 stdin |
| 建 `--type cmd` 会话 | 快码不支持，会话建不起来 | 只用 `desktop`，命令行在桌面里开 |
| 用 `desktop type` 输入 | 被远端中文输入法截胡，命令变乱码 | 一律 `desktop paste` |
| 上传时 `--remote` 写目标文件名 | 报「本地文件不存在」，误判成权限问题 | `--remote` 必须是**已存在的目录**，`--local` 必须**绝对路径** |
| 同一个 transfer_id 重发 | 明明传成功却报 failed | 换远端文件名重新发起 |
| 只看「端口有监听」就说服务正常 | 监听的是别人的程序 | 查 PID 对应命令行，确认是不是你的程序 |
| 改完配置没生效 | 改的是副本，不是运行中的那份 | 先 `netstat` 拿 PID，再定位真实安装目录 |

## 前置条件

- **向日葵账号已登录**：`awesun-cli login status` 为已登录（首次 `awesun-cli login --url` 扫码）。
- **安装 CLI**：`npm install -g @aweray/awesun-cli`（需要 Node.js 18+）。
- **目标机在线**且已开启向日葵；你能从机主处拿到**识别码 + 验证码**。
- **目标机是 Windows**（键鼠坐标、`.bat`/`chcp 65001`、`.ps1` BOM 这些细节都按 Windows 写）。
- 一个支持 `SKILL.md` 规范的 Agent 环境（WorkBuddy、Claude Code 等）。

## 安装

### 方式一：一行命令（推荐，落到 `~/.workbuddy/skills`）

Git Bash / macOS / Linux：

```bash
git clone --depth 1 https://github.com/Object92/sunlogin-remote-ops.git /tmp/sro \
  && rm -rf ~/.workbuddy/skills/sunlogin-remote-ops \
  && cp -r /tmp/sro ~/.workbuddy/skills/sunlogin-remote-ops \
  && rm -rf ~/.workbuddy/skills/sunlogin-remote-ops/.git
```

Windows PowerShell：

```powershell
git clone --depth 1 https://github.com/Object92/sunlogin-remote-ops.git "$env:TEMP\sro"; `
New-Item -ItemType Directory -Force "$env:USERPROFILE\.workbuddy\skills" | Out-Null; `
Remove-Item "$env:USERPROFILE\.workbuddy\skills\sunlogin-remote-ops" -Recurse -Force -ErrorAction SilentlyContinue; `
Copy-Item "$env:TEMP\sro" "$env:USERPROFILE\.workbuddy\skills\sunlogin-remote-ops" -Recurse; `
Remove-Item "$env:USERPROFILE\.workbuddy\skills\sunlogin-remote-ops\.git" -Recurse -Force
```

### 方式二：skills CLI（多 Agent 环境通用）

```bash
npx -y skills add Object92/sunlogin-remote-ops -g -s sunlogin-remote-ops -y
```

`-g` 装到用户级目录，`-s` 指定 skill 名，`-y` 跳过确认。想先看看仓库里有什么，用 `-l` 代替：

```bash
npx -y skills add Object92/sunlogin-remote-ops -l
```

> 两种方式的区别：方式一固定装到 `~/.workbuddy/skills/`，对 WorkBuddy 生效；方式二由 `skills` CLI 决定落到哪个 Agent 目录（Claude Code、Cursor 等），支持多 Agent 时会自动链接。

## 使用

装好后不需要手动调用，用自然语言触发即可：

```
向日葵识别码 123456789，验证码 abc123，帮我连上去看看为什么接口 500
```

```
远程帮我改一下那台机器的配置，识别码 xxx，验证码 xxx
```

Agent 会按 `SKILL.md` 的流程走：先确认登录态与设备在线 → 建桌面会话 → 截图看清现场 → 确认机器身份 → 只读取证 → 备份后改动 → 自测 + 外部验证 → 清理并断开会话。

## 目录结构

```
sunlogin-remote-ops/
├── SKILL.md                      # 主文件：触发条件 / 输入 / 9 步流程 / 异常处理 / 注意事项
├── references/
│   ├── cli-cookbook.md           # awesun-cli 实测命令速查
│   └── hosts.example.md          # 站点清单模板（复制为 hosts.local.md 后填写）
├── README.md
├── LICENSE
└── .gitignore                    # 忽略 hosts.local.md，防止内网信息进版本库
```

## 关于安全

- 本仓库**不含任何真实凭证**：没有识别码、没有验证码、没有内网 IP、没有口令。
- 你自己的机器清单请写进 `references/hosts.local.md`，该文件已被 `.gitignore` 忽略。
- 提交前建议自查：`grep -rnE "密码|password|10\.[0-9]+\.[0-9]+\.[0-9]+" .`
- 验证码是一次性凭证，用完即弃，不要写进任何文件或日志。

## License

MIT
