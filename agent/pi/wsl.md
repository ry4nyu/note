# WSL 安装与配置指南

以 WSL 2 为准（Windows 10 2004+/11）。PowerShell 命令需要管理员权限的会单独标注。

## 1. 环境要求

- Windows 10 版本 2004 及以上（内部版本 19041+），或 Windows 11；64 位。
- BIOS/UEFI 中开启虚拟化：Intel 为 VT-x，AMD 为 SVM。
- 验证方式二选一：
  - 任务管理器 → 性能 → CPU，显示"虚拟化: 已启用"。
  - `(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled` 为 `True`。

## 2. 安装

### 2.1 一键安装（推荐）

```powershell
wsl --install
```

自动启用所需功能、安装 WSL 2 和默认 Ubuntu 发行版。完成后**重启电脑**，重启后会自动打开 Ubuntu 窗口，按提示设置用户名和密码（输入密码不回显字符，属正常）。

> **装不上基本都是网络问题**：WSL 下载器走 WinINet/WinHTTP，浏览器能上网不代表它能下载。代理必须覆盖 WinHTTP/WinINet（推荐开代理软件的 TUN 模式），关掉代理反而装不上。详见 8.2。

### 2.2 安装指定发行版

```powershell
wsl --list --online          # 查看可用发行版
wsl --install -d Ubuntu-26.04
```

加 `--no-launch` 表示只安装、不立即启动。

### 2.3 手动安装（`wsl --install` 不可用）

老系统上 `wsl --install` 会报"无效的命令行选项"，用 DISM 手动启用功能：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

重启后安装 WSL 内核更新包（搜索"WSL2 Linux kernel update package for x64"），执行 2.4 的设置命令，再从 Microsoft Store 安装 Ubuntu。

### 2.4 设为 WSL 2 并验证

```powershell
wsl --set-default-version 2
wsl -l -v                     # VERSION 列应为 2
wsl --set-version Ubuntu 2    # 仍是 1 的发行版手动转换
```

看到发行版、VERSION 为 2、STATE 为 Running 或 Stopped 即安装成功；进入 Linux 后 `uname -a` 可确认内核。初始化：

```bash
sudo apt update && sudo apt upgrade -y
```

## 3. 常用命令

| 命令 | 作用 |
| --- | --- |
| `wsl` | 进入默认发行版 |
| `wsl -d <名称>` | 进入指定发行版 |
| `wsl -l -v` | 查看已安装发行版和状态 |
| `wsl --shutdown` | 关闭所有 WSL 实例（改配置后必做） |
| `wsl --update` | 更新 WSL 内核 |
| `wsl --unregister <名称>` | 删除发行版（**清空其中数据**） |

## 4. Windows 与 WSL 之间的文件

WSL 自动挂载 Windows 磁盘，多数情况不需要"传输"，复制路径即可。

**WSL 访问 Windows**：C 盘是 `/mnt/c`，D 盘是 `/mnt/d`，以此类推。

```bash
ls /mnt/c/Users
cp /mnt/c/Users/ryan/Downloads/a.zip ~/      # Windows → WSL
cp ~/a.zip /mnt/d/tmp/                       # WSL → Windows
```

**Windows 访问 WSL**：资源管理器地址栏输入 `\\wsl.localhost\<发行版名>`（旧写法 `\\wsl$\<发行版名>`），例如 `\\wsl.localhost\Ubuntu\home\ryan`。

**在 WSL 里用资源管理器打开当前目录**：

```bash
cd ~/project && explorer.exe .
```

注意：

- 跨 `/mnt/c` 读写比 Linux 原生目录（`~/`）慢很多：代码放 `~/`，只把要交换的文件拷过去。
- 不要长期用 Windows 侧工具直接编辑 `~/` 里的文件，用 VS Code 的 `code .` 走 WSL 远程。
- `\\wsl.localhost\...` 需要 WSL 正在运行，`wsl --shutdown` 后会断连。

## 5. 网络、DNS 与代理

先分清两个配置文件：`.wslconfig` 在 Windows 用户目录（`%USERPROFILE%\.wslconfig`），管所有发行版；`/etc/wsl.conf` 在发行版内部，只管当前发行版。**两者改完都要 `wsl --shutdown` 再重进才生效。**

### 5.1 网络模式

**默认 NAT 模式**：WSL 2 有独立虚拟网卡和 IP（每次重启可能变），与 Windows 是 NAT 关系。

- Windows → WSL：WSL 2 自带 localhost 转发，浏览器直接开 `http://localhost:8080` 即可。兜底查 IP：`wsl hostname -I`。
- WSL → Windows：用默认网关地址，即 `ip route show default | awk '{print $3}'`。
- 局域网其他机器 → WSL：默认不通，需端口转发 + 放行防火墙（**管理员** PowerShell）：

  ```powershell
  wsl hostname -I          # 先拿 WSL IP，例如 172.20.10.5
  netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=172.20.10.5
  New-NetFirewallRule -DisplayName "WSL 8080" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow
  ```

  删除：`netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0`。WSL IP 变了要重新加，很烦——能用镜像模式就用镜像模式。

**镜像模式（推荐，Windows 11 22H2+ / WSL 2.0.0+）**：在 `%USERPROFILE%\.wslconfig` 写

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
```

WSL 与 Windows 共享网络接口：localhost 双向互通、IPv6 可用、VPN 兼容性更好，**不需要端口转发**，也能自动继承 Windows 系统代理。

### 5.2 DNS

默认每次启动自动生成 `/etc/resolv.conf`，指向 Windows 的 DNS（NAT 模式下是虚拟网关地址）。连公司 VPN 或内网域名解析不了时按顺序试：

1. `.wslconfig` 里加 `dnsTunneling=true`（见上）。
2. 还不行就关闭自动生成，自己维护：

   ```ini
   [wsl2]
   generateResolvConf=false
   ```

   然后在发行版内写 `/etc/resolv.conf`，例如 `nameserver 10.0.0.2`。发行版重启后可能被覆盖，改完确认一下。

### 5.3 代理

- 只有「镜像模式 + `autoProxy=true`」会自动继承 Windows 代理，NAT 模式不会。
- 手动配置（写进 `~/.bashrc` 或 `/etc/environment`）：**NAT 模式填 Windows 主机 IP，镜像模式填 `127.0.0.1`**。

  ```bash
  export http_proxy=http://<主机IP>:7890
  export https_proxy=$http_proxy
  export no_proxy=localhost,127.0.0.1
  ```

- `sudo` 不继承代理，需要时 `sudo -E <命令>`；`apt` 单独配 `/etc/apt/apt.conf.d/95proxy`：

  ```
  Acquire::http::Proxy "http://<主机IP>:7890";
  Acquire::https::Proxy "http://<主机IP>:7890";
  ```

- 验证：`curl -I https://www.google.com`、`env | grep -i proxy`。

## 6. 禁止在 WSL 中调用 Windows 程序

WSL 默认开启 interop，并把 Windows 的 `PATH` 追加到 Linux `PATH` 后面。结果是 Linux 里能直接跑 `notepad.exe`、`explorer.exe .`，`which code` 可能指向 Windows 的 exe，补全和脚本里也容易混进 `.exe`。

关掉它——编辑发行版内 `/etc/wsl.conf`（需要 sudo，没有就新建）：

```ini
[interop]
enabled = false
appendWindowsPath = false
```

两项可单独使用：

| 配置 | 效果 |
| --- | --- |
| `appendWindowsPath = false` | 只清理 `PATH`，interop 仍可用：绝对路径 `/mnt/c/Windows/explorer.exe .` 还能调，`code .` 这类靠 Windows PATH 的命令会失效 |
| `enabled = false` | 彻底禁止执行 `.exe`，`explorer.exe` 也会失败 |

选 `enabled = false` 前想清楚副作用：

- 所有 `.exe` 调用失效，包括依赖 interop 的工具（Docker Desktop 集成、从 Linux 启动 Windows GUI 程序等）。
- `code .` / `code-insiders .` 用不了；VS Code Remote-WSL 不受影响（它由 Windows 侧 `wsl.exe` 拉起，走远程连接，不是 interop）。
- 改完必须 `wsl --shutdown` 再重进才生效。

验证：

```bash
echo $PATH                                   # 不应出现 /mnt/c/...
which notepad.exe                            # 应该没有输出
cat /proc/sys/fs/binfmt_misc/WSLInterop      # enabled = false 时显示 disabled
```

**仅临时禁用**（重启发行版即恢复，适合跑一次干净测试）：

```bash
sudo sh -c 'echo 0 > /proc/sys/fs/binfmt_misc/WSLInterop'
```

## 7. 在 Linux 中配置环境变量

先想清楚两件事：**给谁用**（当前 shell / 所有用户 / 某个服务）、**什么时候读**（登录时 / 每次开 shell / 只在脚本里）。选错位置是"改了没生效"的唯一原因。

### 7.1 放哪里

| 位置 | 生效范围 | 生效时机 | 适用场景 |
| --- | --- | --- | --- |
| `export FOO=1` | 当前 shell 及其子进程 | 立即 | 临时测试 |
| `~/.bashrc` | 当前用户 | 每次打开交互式 shell | 最常用：PATH、别名 |
| `~/.profile` | 当前用户 | 登录时（含 `wsl` 启动、SSH 登录） | 只该设一次的东西 |
| `/etc/environment` | 所有用户 | 登录时，由 PAM 读取 | 全机统一变量；**只能写 `KEY=VALUE`，不支持 `$PATH` 和 `export`** |
| `/etc/profile.d/foo.sh` | 所有用户 | 登录时 | 需要逻辑或 `$PATH` 拼接的全机变量 |

WSL 里打开的终端通常是交互式登录 shell，`~/.bashrc` 和 `~/.profile` 都会读，所以写在 `~/.bashrc` 最省事：

```bash
# 追加到 ~/.bashrc 末尾
export EDITOR=vim
export PATH="$HOME/.local/bin:$PATH"   # $PATH 放后面，别把系统路径盖掉
```

```bash
source ~/.bashrc      # 立刻生效，或直接关掉重开终端
```

### 7.2 验证

```bash
echo $EDITOR                     # 看单个变量
env | grep -i proxy              # 看一类变量
printenv PATH | tr ':' '\n'      # 按行看 PATH 每一项
bash -lc 'echo $EDITOR'          # 模拟登录 shell，确认写对了文件
bash -c 'echo $EDITOR'           # 非交互 shell 不读 ~/.bashrc，为空属正常
```

### 7.3 三个常见坑

- **`sudo` 不继承**：环境变量会被清掉，需要时用 `sudo -E <命令>`（`sudoers` 里 `env_reset` 限制的变量仍会丢弃）。系统级变量写 `/etc/environment` 更稳。
- **非交互场景读不到 `~/.bashrc`**：cron、systemd 服务、`ssh host 'cmd'` 都不走交互式 shell。systemd 用 unit 里的 `Environment=FOO=1` 或 `EnvironmentFile=/etc/foo.env`；cron 直接在 crontab 顶部写 `FOO=1`。
- **PATH 顺序写反**：`export PATH="$PATH:$HOME/.local/bin"` 会让同名程序继续用系统那个；想优先用自己的，新路径必须放前面。

### 7.4 与 Windows 环境变量互通

WSL 默认把 Windows 的 `PATH` 追加到 Linux `PATH` 后面（见第 6 节关掉它），其他 Windows 变量**不会**自动进来。需要传的话在 Windows 侧用 `WSLENV`：

```powershell
# /p 表示路径，/l 表示只往 Linux 传
$env:WSLENV = "FOO/p"
```

多数项目里更简单的做法是各写各的：Linux 配置放 `~/.bashrc`，Windows 配置放系统环境变量，互不依赖。

## 8. 常见问题

### 8.1 `HCS_E_HYPERV_NOT_INSTALLED`（"此计算机上未启用虚拟化"）

典型现象：`wsl --install -d Ubuntu-26.04` 下载、安装都正常，最后创建虚拟机时失败，并提示"运行 `wsl.exe --install --no-distribution` 启用虚拟机平台"。**照着提示去装 Windows 功能是无效的**，真实原因在 BIOS。

三条命令定位卡在哪一层：

```powershell
(Get-CimInstance Win32_Processor).VMMonitorModeExtensions          # CPU 是否支持虚拟化
(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled    # BIOS 是否把虚拟化交给系统
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent           # 是否有 Hypervisor 在跑
```

| 现象 | 结论 |
| --- | --- |
| `True` / `False` / `False` | **BIOS 里 SVM/VT-x 没开**，最常见 |
| `True` / `True` / `True` | 已经好了，不必再管虚拟化 |
| 三项都 `True` 但仍报错 | Windows 功能或 VBS 冲突，见第 2 步 |
| `False` / `True` / `True` | **正常**：hypervisor 已接管，root partition 看不到原始虚拟化扩展，这是启用成功的标志 |
| `False` / `False` / `False` | CPU/主板不支持，或虚拟化被厂商锁死 |

修复顺序不能颠倒：

1. **进 BIOS 开 SVM**（AMD）/ VT-x（Intel）。ASUS AMI BIOS 一般在 `Advanced` → `CPU Configuration` → `SVM Mode` → `Enabled`；找不到就在 `Advanced` → `AMD CBS` → `CPU Common Options` → `SVM`。顺手打开 `IOMMU`、`Above 4G Decoding`。
2. **重启回系统后**确认 `VirtualizationFirmwareEnabled` 变 `True`，执行 `wsl --install --no-distribution`（一条命令启用全部所需组件），**再重启一次**。
3. 最后 `wsl --set-default-version 2`，再 `wsl --install -d Ubuntu-26.04`。

做完第 1 步仍失败，按命中率排查：

- **内存完整性**（Windows 安全中心 → 设备安全性 → 内核隔离）开着会和 SVM 抢 VBS 权限，关掉重启。
- VMware / VirtualBox / 安卓模拟器 / 国产安全驱动占用了虚拟化层，先卸载。
- WSL 版本太旧（`wsl --update`）。Insider 通道的 Windows + WSL 组合更容易出这个问题。
- 别把 BIOS 里的 SVM 和杀毒软件那个 SVM 搞混。

> 案例：AMD Ryzen 7 5700X3D + ASUS AMI BIOS，Windows 11 26300，WSL 3.0.3.0 已装、默认版本已是 2，装 Ubuntu-26.04 报此错。诊断为 `True / False / False`，即 BIOS 未开 SVM，与 Windows 功能无关。

### 8.2 `WININET_E_INVALID_SERVER_RESPONSE` / 卡在"正在下载"

虚拟化已就绪（`VirtualizationFirmwareEnabled=True`、`HypervisorPresent=True`），但 `wsl -l -v` 报"没有已安装的分发"——问题在**下载通道被墙**，不在虚拟化。

判据（三条都命中即可确定是网络）：

```powershell
# 1. 短链被劫持或失效：应跳到微软/WSL 页面，跳到 bing.com 就是失败
curl.exe -I -L "https://aka.ms/wslubuntu2604" | Select-String -Pattern 'Location'
# 2. 微软分发 CDN 的 DNS 被污染（解析为空）
Resolve-DnsName delivery.mp.microsoft.com
# 3. WinHTTP 无代理，但系统代理开着
netsh winhttp show proxy
```

WSL 下载器与浏览器/`Invoke-WebRequest` 走不同网络栈，所以会出现"浏览器能上网，`wsl --install` 就是下载不了"。

修复，按成功率排序：

1. **代理软件开 TUN 模式**（需管理员）：接管全部流量和 DNS 解析，污染域名也能解出来。然后 `wsl --install -d Ubuntu-26.04 --no-launch`。一次成功率最高。
2. **给 WinHTTP 也配代理**（管理员），用完还原：

   ```powershell
   netsh winhttp set proxy 127.0.0.1:10808
   wsl --install -d Ubuntu-26.04 --no-launch
   netsh winhttp reset proxy
   ```

   端口要填代理软件的 **HTTP 入站**端口。v2rayN 默认 10808=SOCKS、10809=HTTP，填错协议同样报 `WININET_E_INVALID_SERVER_RESPONSE`。
3. **兜底：手动下载镜像再导入**，完全不依赖微软通道：

   ```powershell
   # 用浏览器走代理下载 https://cdimages.ubuntu.com/ubuntu-wsl/<代号>/ 下的 rootfs.tar.gz
   # 26.04 = resolute，24.04 = noble
   wsl --import Ubuntu-26.04 D:\WSL\Ubuntu-26.04 <下载的rootfs.tar.gz> --version 2
   wsl -d Ubuntu-26.04
   ```

   `--import` 出来的发行版不出现在 `wsl --install` 列表里，首次启动是 root（需自己 `adduser`）。

排查代理时先分清端口协议：

```powershell
Get-NetTCPConnection -State Listen -LocalPort 10808,10809 | ForEach-Object {
  "port $($_.LocalPort) <- $((Get-Process -Id $_.OwningProcess).Name)" }
```

> 案例：同机开 SVM 后 `VirtualizationFirmwareEnabled`、`HypervisorPresent` 均为 `True`，而 `VMMonitorModeExtensions`/`SLAT` 变 `False`（正常现象）。`wsl --list --online` 能出列表，但 `wsl --install -d Ubuntu-26.04` 卡 10 分钟无进展，加 `--web-download` 后报 `Wsl/InstallDistro/WININET_E_INVALID_SERVER_RESPONSE`；同时 `aka.ms/wslubuntu2604` 跳 bing.com、`delivery.mp.microsoft.com` 解析为空、WinHTTP 显示 Direct。根因是代理未覆盖 WinINet/WinHTTP。

### 8.3 `WSL 2 requires an update to its kernel component`

执行 `wsl --update`，然后 `wsl --shutdown` 重启 WSL。

### 8.4 `wsl --install` 无反应或报"无效的命令行选项"

Windows 版本过低：参考 2.3 手动安装，或先更新 Windows。

### 8.5 安装卡在 0% 或长时间无进展

不是"稍等就好"。先确认是否卡在下载：`wsl -l -v` 若无发行版，多半是代理未覆盖 WinHTTP/WinINet，按 8.2 排查。

### 8.6 想换安装位置

WSL 数据默认在 `%LOCALAPPDATA%\Packages\` 下。用 `wsl --export <名称> <tar路径>` 导出，再 `wsl --import <名称> <安装目录> <tar路径>` 导入到其他盘。
