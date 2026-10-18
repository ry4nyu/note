# WSL 开发环境配置

Windows 11 + WSL 2。装完后**有两件事必须配**，不配后面全是坑：镜像模式网络、禁止 WSL 调 Windows 程序。

## 0. 检查清单

- [ ] BIOS 虚拟化已开，`wsl -l -v` 的 VERSION 是 `2`
- [ ] `%USERPROFILE%\.wslconfig` 写了 `networkingMode=mirrored`
- [ ] `/etc/wsl.conf` 写了 `[interop] enabled = false` / `appendWindowsPath = false`
- [ ] 改完执行过 `wsl --shutdown`，再重进
- [ ] 两条验证通过（2.3、3.4）

## 1. 安装

### 1.1 要求

- Windows 11（或 Win10 2004+/19041+），64 位。
- BIOS/UEFI 开启虚拟化：Intel VT-x / AMD SVM。任务管理器 → 性能 → CPU 显示"虚拟化: 已启用"。

### 1.2 一键安装

```powershell
wsl --install                  # 装 WSL 2 + 默认 Ubuntu，完成后重启
wsl --install -d Ubuntu-26.04  # 指定发行版，--no-launch 只装不启动
wsl --list --online            # 看有哪些
```

重启后自动弹出 Ubuntu 窗口，按提示建用户名密码（输密码不回显，正常）。

> 装不上基本都是网络问题，不是虚拟化问题——见 5.2。WSL 下载器走 WinINet/WinHTTP，浏览器能上网不代表它能下载。

### 1.3 确认是 WSL 2

```powershell
wsl -l -v                      # VERSION 列应为 2，STATE 为 Running/Stopped
wsl --set-default-version 2
wsl --set-version Ubuntu 2     # 仍是 1 的手动转换
```

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wl-clipboard
uname -a                       # 确认内核
```

### 1.4 `wsl --install` 报"无效的命令行选项"

系统太老，手动启用功能后重启（**管理员** PowerShell）：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

然后装 WSL2 内核更新包（搜"WSL2 Linux kernel update package for x64"），回到 1.3。

## 2. 必配一：镜像模式

### 2.1 为什么必须配

默认 NAT 模式下 WSL 是独立虚拟网卡 + 私有 IP，带来三个固定麻烦：

- Windows 侧的代理软件只监听 `127.0.0.1`，WSL 看不到 → 每次启动报"检测到 localhost 代理配置，但未镜像到 WSL"，`http_proxy` 为空，一切下载都失败。
- 局域网其他机器访问 WSL 要手动端口转发 + 放行防火墙，IP 一变就得重配。
- 连 VPN 时路由/DNS 容易坏。

镜像模式让 WSL 和 Windows 共用网络接口，以上全部消失。

### 2.2 配置

新建或编辑 `%USERPROFILE%\.wslconfig`（Windows 用户目录，管所有发行版）：

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
```

**当前机器上的实际配置就是这样，已验证可用**（代理 xray 监听 `127.0.0.1:10808`，WSL 内自动拿到 `http_proxy`）。

生效：

```powershell
wsl --shutdown      # 会关掉所有 WSL 会话
```

### 2.3 验证

```bash
env | grep -i proxy                                   # 应有 http_proxy=http://127.0.0.1:10808
ip -4 addr | grep -c 'inet 172\.'                     # 应为 0（没有 NAT 私网地址了）
curl -sS -o /dev/null -w '%{http_code}\n' https://www.google.com   # 200
```

启动时不再出现 localhost 代理警告，即成功。

### 2.4 镜像模式的副作用

- WSL 的 `127.0.0.1` 与 Windows 同源：**Windows 已占用的端口，WSL 里绑不上**。真需要就配 `ignoredPorts=3000,9090`（镜像模式专用）。
- 反过来 WSL 里起的服务，Windows 侧 `localhost:端口` 直接能访问，不用再 `localhostForwarding` 折腾。

### 2.5 备选：consomme 模式

镜像模式出问题（端口冲突、VPN、防火墙误拦）时换它，WSL 2.9.3+：

```ini
[wsl2]
networkingMode=consomme
virtio=true
```

用户态 NAT，行为仍属 NAT 一类：localhost 转发有，局域网访问仍需端口转发。旧名 `virtioproxy` 也认。

| 模式 | 网络身份 | Windows → WSL | 局域网 → WSL | 继承系统代理 |
| --- | --- | --- | --- | --- |
| `nat`（默认） | 私有 IP | localhost 转发 | 端口转发 | 否 |
| `mirrored`（推荐） | 共用 Windows 网卡 | localhost 全通 | 通 | `autoProxy=true` 自动 |
| `consomme` | 私有 IP（用户态 NAT） | localhost 转发 | 端口转发 | 否 |
| `none` | 无网络 | — | — | — |

> `networkingMode=nat` 失败时 WSL 会自动回落到 consomme（2.3.25+），所以"NAT 模式"下的怪行为先确认实际跑的哪个。

## 3. 必配二：禁止 WSL 调用 Windows 程序

### 3.1 为什么

WSL 默认开启 interop，并把 Windows `PATH` 追加到 Linux `PATH` 后面。后果：Linux 里能跑 `notepad.exe`、`explorer.exe .`，`which code` 可能指向 Windows 的 exe，补全和脚本里混进 `.exe`，PATH 里多出几十条 `/mnt/c/...`。

### 3.2 配置

编辑发行版内 `/etc/wsl.conf`（需要 sudo；**保留已有内容**，不要覆盖 `[boot]`/`[user]`）：

```ini
[interop]
enabled = false
appendWindowsPath = false
```

**当前机器上的实际配置**（改前备份在 `/etc/wsl.conf.bak`）：

```ini
[boot]
systemd=true

[user]
default=ryanyu

[interop]
enabled = false
appendWindowsPath = false
```

两项可单独用：

| 配置 | 效果 |
| --- | --- |
| `appendWindowsPath = false` | 只清 `PATH`，interop 仍可用：`/mnt/c/Windows/explorer.exe .` 还能调，`code .` 这类靠 Windows PATH 的失效 |
| `enabled = false` | 彻底禁止执行 `.exe` |

### 3.3 副作用

- 所有 `.exe` 调用失效：`explorer.exe .`、`code .`、从 Linux 启动 Windows GUI 程序、Docker Desktop 集成。
- VS Code Remote-WSL **不受影响**（Windows 侧 `wsl.exe` 拉起远程连接，不走 interop）。
- 想用 `code .` 或 Docker Desktop 集成，就把这两行删掉。
- enabled = false 有个缺点：会导致wsl中的claude启动很慢（6s), rg 无效

### 3.4 验证

```bash
echo $PATH | tr ':' '\n' | grep -c '^/mnt/c'   # 0
command -v notepad.exe                         # 空
notepad.exe                                    # 报 UtilAcceptVsock ... accept4 failed
```

`cat /proc/sys/fs/binfmt_misc/WSLInterop` 在 WSL 3.x 下仍显示 `enabled`，**别拿它判断**。

仅在本次会话临时禁用（重启发行版即恢复）：

```bash
sudo sh -c 'echo 0 > /proc/sys/fs/binfmt_misc/WSLInterop'
```

## 4. 日常操作

### 4.1 常用命令

| 命令 | 作用 |
| --- | --- |
| `wsl` / `wsl -d <名称>` | 进入默认 / 指定发行版 |
| `wsl -l -v` | 看发行版和状态 |
| `wsl --shutdown` | 关闭所有实例（**改任何配置后必做**） |
| `wsl --update` | 更新 WSL 内核 |
| `wsl --unregister <名称>` | 删除发行版（**清空数据**） |
| `wsl -u root -e bash -c '...'` | 免密执行 root 命令 |

### 4.2 文件互访

WSL 访问 Windows：C 盘是 `/mnt/c`。

```bash
ls /mnt/c/Users
cp /mnt/c/Users/ryan/Downloads/a.zip ~/
```

Windows 访问 WSL：资源管理器输入 `\\wsl.localhost\<发行版名>`，例如 `\\wsl.localhost\Ubuntu-26.04\home\ryanyu`。需要 WSL 正在运行，`wsl --shutdown` 后会断连。

- 跨 `/mnt/c` 读写比 `~/` 慢很多：代码放 `~/`，只拷要交换的文件。
- 别用 Windows 侧工具直接编辑 `~/` 里的文件，用 VS Code 的 Remote-WSL。

### 4.3 环境变量

选错位置是"改了没生效"的唯一原因。WSL 终端通常是交互式登录 shell，`~/.bashrc` 和 `~/.profile` 都会读，写 `~/.bashrc` 最省事。

| 位置 | 生效范围 | 时机 |
| --- | --- | --- |
| `export FOO=1` | 当前 shell | 立即，临时测试 |
| `~/.bashrc` | 当前用户 | 每次开交互式 shell（最常用） |
| `~/.profile` | 当前用户 | 登录时 |
| `/etc/environment` | 所有用户 | 登录时，**只能写 `KEY=VALUE`** |
| `/etc/profile.d/foo.sh` | 所有用户 | 登录时，可写逻辑和 `$PATH` |

```bash
# 追加到 ~/.bashrc 末尾
export EDITOR=vim
export PATH="$HOME/.local/bin:$PATH"   # 新路径放前面才优先
source ~/.bashrc
```

三个坑：

- `sudo` 不继承变量 → `sudo -E <命令>`；系统级写 `/etc/environment`。
- cron / systemd / `ssh host 'cmd'` 不走交互式 shell，读不到 `~/.bashrc`。
- `export PATH="$PATH:新路径"` 会把优先级搞反。

Windows 的 `PATH` 已在第 3 节关掉；其他变量要传用 `WSLENV`（`$env:WSLENV = "FOO/p"`），但多数项目各写各的更简单。

### 4.4 代理与 DNS

- 镜像模式 + `autoProxy=true`：`http_proxy`/`https_proxy`/`no_proxy` 自动注入发行版，不用手写。
- `sudo` 不继承代理：`sudo -E apt update`；`apt` 也可单独配 `/etc/apt/apt.conf.d/95proxy`：

  ```
  Acquire::http::Proxy "http://127.0.0.1:10808";
  Acquire::https::Proxy "http://127.0.0.1:10808";
  ```

- DNS 解析不了内网域名：先确认 `.wslconfig` 里有 `dnsTunneling=true`（已配）。还不行就 `generateResolvConf=false` 并自己维护 `/etc/resolv.conf`。
- Windows 侧改代理端口/开关后，WSL 里执行 `wsl --shutdown` 再进才会重新镜像。

## 5. 排错

### 5.1 `HCS_E_HYPERV_NOT_INSTALLED`（"此计算机上未启用虚拟化"）

现象：下载安装都正常，创建虚拟机时失败，提示去运行 `wsl.exe --install --no-distribution`。**照着提示装 Windows 功能是无效的**，真实原因在 BIOS。

```powershell
(Get-CimInstance Win32_Processor).VMMonitorModeExtensions          # CPU 支持虚拟化
(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled    # BIOS 交给了系统
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent           # 有 Hypervisor 在跑
```

| 现象 | 结论 |
| --- | --- |
| `True` / `False` / `False` | **BIOS 里 SVM/VT-x 没开**，最常见 |
| `False` / `True` / `True` | **正常**：hypervisor 已接管，root partition 看不到原始扩展 |
| `True` / `True` / `True` | 已就绪，别再管虚拟化 |
| 都 `True` 仍报错 | Windows 功能或 VBS 冲突，见下 |
| 都 `False` | CPU/主板不支持，或厂商锁死 |

修复顺序不能颠倒：

1. 进 BIOS 开 SVM（AMD）/ VT-x（Intel）：ASUS AMI 一般在 `Advanced` → `CPU Configuration` → `SVM Mode` → `Enabled`，或 `Advanced` → `AMD CBS` → `CPU Common Options` → `SVM`。顺手开 `IOMMU`、`Above 4G Decoding`。
2. 重启回系统后确认 `VirtualizationFirmwareEnabled` 变 `True`，执行 `wsl --install --no-distribution`，**再重启一次**。
3. `wsl --set-default-version 2`，再装发行版。

仍失败按命中率排查：关掉内存完整性（内核隔离）→ 卸载 VMware/VirtualBox/安卓模拟器/国产安全驱动 → `wsl --update`。

> 案例：AMD Ryzen 7 5700X3D + ASUS AMI BIOS，Win11 26300，WSL 已装且默认版本为 2，装 Ubuntu-26.04 报此错。诊断 `True / False / False`，即 BIOS 未开 SVM，与 Windows 功能无关。

### 5.2 `WININET_E_INVALID_SERVER_RESPONSE` / 卡在"正在下载"

虚拟化已就绪（`VirtualizationFirmwareEnabled=True`、`HypervisorPresent=True`）但 `wsl -l -v` 说没有发行版——问题在**下载通道**，不在虚拟化。

判据（三条都命中即可确定是网络）：

```powershell
curl.exe -I -L "https://aka.ms/wslubuntu2604" | Select-String 'Location'   # 跳 bing.com 就是失败
Resolve-DnsName delivery.mp.microsoft.com                                  # 解析为空 = DNS 被污染
netsh winhttp show proxy                                                   # Direct = WinHTTP 无代理
```

WSL 下载器走 WinINet/WinHTTP，和浏览器不是一条网络栈，所以"浏览器能上网，`wsl --install` 下不动"很常见。

修复，按成功率排序：

1. **代理软件开 TUN 模式**（管理员），接管全部流量和 DNS，然后：
   ```powershell
   wsl --install -d Ubuntu-26.04 --no-launch
   ```
2. **给 WinHTTP 也配代理**（管理员），用完还原：
   ```powershell
   netsh winhttp set proxy 127.0.0.1:10808
   wsl --install -d Ubuntu-26.04 --no-launch
   netsh winhttp reset proxy
   ```
   端口要填代理软件的 **HTTP 入站**端口，填成 SOCKS 端口同样报这个错。查端口归属：
   ```powershell
   Get-NetTCPConnection -State Listen -LocalPort 10808,10809 |
     ForEach-Object { "port $($_.LocalPort) <- $((Get-Process -Id $_.OwningProcess).ProcessName)" }
   ```
3. **兜底：手动下载镜像再导入**，完全不依赖微软通道：
   ```powershell
   # 浏览器走代理下载 https://cdimages.ubuntu.com/ubuntu-wsl/<代号>/ 下的 rootfs.tar.gz
   # 26.04 = resolute，24.04 = noble
   wsl --import Ubuntu-26.04 D:\WSL\Ubuntu-26.04 <rootfs.tar.gz> --version 2
   wsl -d Ubuntu-26.04
   ```
   `--import` 出来的不在 `wsl --install` 列表里，首次启动是 root，需自己 `adduser`。

> 案例：`wsl --list --online` 能出列表，`wsl --install -d Ubuntu-26.04` 卡 10 分钟无进展；加 `--web-download` 报 `Wsl/InstallDistro/WININET_E_INVALID_SERVER_RESPONSE`；同时 `aka.ms/wslubuntu2604` 跳 bing.com、`delivery.mp.microsoft.com` 解析为空、WinHTTP 显示 Direct。根因是代理未覆盖 WinInet/WinHTTP。

### 5.3 其他

- `WSL 2 requires an update to its kernel component` → `wsl --update`，然后 `wsl --shutdown`。
- 安装卡 0% 或长时间无进展 → 先 `wsl -l -v`，没有发行版就是下载被卡，按 5.2 处理，不是"再等等"。
- 想换安装位置 → `wsl --export <名称> <tar>`，再 `wsl --import <名称> <目录> <tar>`；数据默认在 `%LOCALAPPDATA%\Packages\`。
- 配置改了不生效 → 十有八九漏了 `wsl --shutdown`。`.wslconfig` 管所有发行版，`/etc/wsl.conf` 只管当前发行版。
