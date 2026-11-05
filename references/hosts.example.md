# 站点清单模板

把这份文件复制为 `references/hosts.local.md`，填入你自己的机器。
`hosts.local.md` 已被 `.gitignore` 忽略，**不会进入版本库**。

> 只记「稳定不变的结构」，**不要记验证码**（一次性凭证）。

## 模板

### <机器别名>

| 项 | 值 |
|---|---|
| 内网 IP | `10.0.0.10` |
| 向日葵设备 | `<设备名，awesun-cli device ls -o json 里的 name>` |
| 对外入口 | `http://10.0.0.10/`（nginx 80） |
| 系统 | Windows 版本；是否开 3389/22；是否有共享 |

#### 端口分配

| 端口 | 谁在用 |
|---|---|
| 80 | nginx |
| 8080 | （写明归属，**别停别人的服务**） |
| 8081 | 本系统后端 |
| 3306 | MySQL |

#### 关键路径

| 用途 | 路径 |
|---|---|
| 发布包根 | `D:\app\` |
| 后端 jar | `D:\app\backend\*.jar`（旧版留 `.jar.bak`） |
| 后端配置 | `D:\app\config\application.yml` |
| 启停脚本 | `D:\app\scripts\start-backend.bat`、`stop-backend.ps1` |
| 日志 | `D:\app\logs\backend.log` |
| nginx 程序 | `D:\nginx\nginx.exe` |
| nginx 配置 | `D:\nginx\conf\nginx.conf` |

#### 验证口径

```bash
# 期望 code == 0（注意不是 200）
curl -s -X POST http://10.0.0.10/api/auth/login \
  -H "Content-Type: application/json" -d '{"username":"<user>","password":"<pwd>"}'
```

#### 已知遗留

- 例如：后端未配置开机自启，重启后需手工拉脚本。

## 填表原则

1. **路径和端口是稳定资产**，值得记；口令、验证码、token 不值得记。
2. 写之前先核实：`netstat -ano | findstr :端口` 拿 PID，再查该 PID 的命令行，别凭印象写。
3. 一个系统一个 `###` 小节，机器多了按内网 IP 排序。
