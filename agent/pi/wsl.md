# WSL 安装指南

WSL（Windows Subsystem for Linux）让你在 Windows 上直接运行 Linux 环境。下面以 WSL 2 为准。

## 1. 环境要求

- Windows 10 版本 2004 及以上（内部版本 19041+），或 Windows 11。
- 64 位系统。
- 在 BIOS/UEFI 中开启虚拟化（Intel 为 VT-x，AMD 为 SVM）。
  - 任务管理器 → 性能 → CPU，若显示“虚拟化: 已启用”即可。
  - 或命令行验证，必须为 `True`：`(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled`

## 2. 一键安装（推荐）

建议先关闭网络代理（梯子），网络不稳定，很容易安装失败.
以**管理员身份**打开 PowerShell 或 Windows 终端，执行：

```powershell
wsl --install
```

该命令会自动启用所需功能、安装 WSL 2 和默认的 Ubuntu 发行版。完成后**重启电脑**。

重启后会自动打开 Ubuntu 窗口，按提示设置 Linux 用户名和密码（输入密码时不显示字符，属正常现象）。

## 3. 安装指定发行版

查看可用的发行版：

```powershell
wsl --list --online
```

安装指定版本，例如 Ubuntu-26.04：

```powershell
wsl --install -d Ubuntu-26.04
```

## 4. 设置 WSL 2 为默认版本

```powershell
wsl --set-default-version 2
```

查看已安装的发行版及其版本（VERSION 列应为 2）：

```powershell
wsl -l -v
```

若某个发行版是 1，可手动转换：

```powershell
wsl --set-version Ubuntu 2
```

## 5. 常用命令

| 命令 | 作用 |
| --- | --- |
| `wsl` | 进入默认发行版 |
| `wsl -d <名称>` | 进入指定发行版 |
| `wsl --list --verbose` | 查看已安装发行版和状态 |
| `wsl --shutdown` | 关闭所有 WSL 实例 |
| `wsl --update` | 更新 WSL 内核 |
| `wsl --unregister <名称>` | 删除发行版（会清空其中的数据，谨慎使用） |

## 6. Windows 与 WSL 之间传输文件

WSL 会自动挂载 Windows 磁盘，所以大部分情况不需要“传输”工具，直接复制路径即可。

**从 WSL 访问 Windows 文件**：Windows 的 C 盘挂载在 `/mnt/c`，D 盘为 `/mnt/d`，以此类推。

```bash
ls /mnt/c/Users
cp /mnt/c/Users/ryan/Downloads/a.zip ~/      # Windows → WSL
cp ~/a.zip /mnt/d/tmp/                       # WSL → Windows
```

**从 Windows 访问 WSL 文件**：在资源管理器地址栏输入 `\\wsl.localhost\<发行版名>`（旧写法 `\\wsl$\<发行版名>`），例如 `\\wsl.localhost\Ubuntu\home\ryan`。

**在 WSL 里用资源管理器打开当前目录**：

```bash
cd ~/project && explorer.exe .
```

**注意**：

- 跨 `/mnt/c` 读写比在 Linux 原生目录（`~/`）慢很多，代码放 `~/` 下，只把需要交换的文件拷过去。
- 不要把 WSL 的 `~/` 当作 Windows 文件来长期编辑（VS Code 用 `code .` 连 WSL 远程即可）。
- Windows 侧访问 `\\wsl.localhost\...` 需要 WSL 正在运行，`wsl --shutdown` 后会断连。

**两台机器之间传**（WSL 有自己的 IP，也可以直接用 WSL 里的 `scp`/`rsync`）：

```bash
scp file.zip user@host:/path/          # 本机 → 远程
rsync -avz ./dir/ user@host:/path/dir/ # 增量同步目录
```

若远端也是 Windows/WSL，可在 Windows 侧开 SSH 服务后用 `scp -P <端口>`，或走 SMB 共享挂载。

## 7. 手动安装（`wsl --install` 不可用时）

Windows 版本较旧时 `wsl --install` 可能不支持，需要手动启用功能，以管理员身份执行：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

重启电脑后，下载并安装 WSL 内核更新包（搜索“WSL2 Linux kernel update package for x64”），再执行第 4 节的设置命令，然后从 Microsoft Store 安装 Ubuntu。

## 8. 常见问题

**1. 提示“未启用虚拟机平台”或“请在 BIOS 中启用虚拟化”**
进入 BIOS 开启 Intel VT-x / AMD SVM，保存后重启。

**1.1 错误码 `HCS_E_HYPERV_NOT_INSTALLED`（“WSL2 无法启动，因为此计算机上未启用虚拟化”）**

典型现象：`wsl --install -d Ubuntu-26.04` 显示“正在下载”“正在安装”都正常，最后创建虚拟机时失败，并附带建议“运行 `wsl.exe --install --no-distribution` 启用虚拟机平台”。**照着提示去装 Windows 功能是无效的**，因为真实原因在 BIOS。

先诊断，三条命令判断卡在哪一层：

```powershell
(Get-CimInstance Win32_Processor).VMMonitorModeExtensions          # CPU 是否支持虚拟化，AMD/Intel 都是 True 就没问题
(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled    # BIOS 是否把虚拟化交给系统，False 就是病根
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent           # 有没有 Hypervisor 在跑，正常启用后为 True
```

判读：

| 现象 | 结论 |
| --- | --- |
| `VMMonitorModeExtensions=True`、`VirtualizationFirmwareEnabled=False`、`HypervisorPresent=False` | **BIOS 里 SVM/VT-x 没开**，最常见 |
| 三项都 True 但仍报错 | Windows 功能或 VBS 冲突，见下面第 2 步 |
| `VMMonitorModeExtensions=False` | CPU/主板不支持，或虚拟化被厂商锁死 |

修复顺序（顺序不能颠倒）：

1. **进 BIOS 开 SVM**（AMD）/ VT-x（Intel）。ASUS AMI BIOS 一般在 `Advanced` → `CPU Configuration` → `SVM Mode` → `Enabled`；找不到就在 `Advanced` → `AMD CBS` → `CPU Common Options` → `SVM`。顺手把 `IOMMU`、`Above 4G Decoding` 也打开。
2. **重启回系统后**验证 `VirtualizationFirmwareEnabled` 已变 `True`，再执行 `wsl --install --no-distribution`（一条命令启用全部所需组件），然后**再重启一次**。
3. 最后 `wsl --set-default-version 2` 并 `wsl --install -d Ubuntu-26.04`。

第 1 步做完仍失败的排查项，按命中率排序：

- **内存完整性**（Windows 安全中心 → 设备安全性 → 内核隔离）开着会和 SVM 抢 VBS 权限，关掉重启。
- VMware / VirtualBox / 安卓模拟器 / 国产安全驱动占用了虚拟化层，先卸载。
- WSL 版本太旧：`wsl --update`。Insider 通道的 Windows 和 WSL 版本组合更容易出这个问题。
- 注意别把 BIOS 里的 SVM 和杀毒软件那个 SVM 搞混。

真实案例：AMD Ryzen 7 5700X3D + ASUS AMI BIOS，Windows 11 26300，WSL 3.0.3.0 已装、默认版本已是 2，装 Ubuntu-26.04 报此错。诊断为 `VMMonitorModeExtensions=True` / `VirtualizationFirmwareEnabled=False` / `HypervisorPresent=False`，即 BIOS 未开 SVM，与 Windows 功能无关。

**2. 提示 `WSL 2 requires an update to its kernel component`**
执行 `wsl --update`，然后重启 WSL：`wsl --shutdown`。

**3. `wsl --install` 没有反应或报错 `无效的命令行选项`**
Windows 版本过低，参考第 7 节手动安装，或先更新 Windows。

**4. 安装时卡在 0%**
检查网络，或稍等几分钟；可以关闭后重新执行 `wsl --install`。

**5. 想换安装位置**
WSL 的数据默认在 `%LOCALAPPDATA%\Packages\` 下。可用 `wsl --export <名称> <tar路径>` 导出，再用 `wsl --import <名称> <安装目录> <tar路径>` 导入到其他盘。

## 9. 验证安装

在 PowerShell 中执行：

```powershell
wsl -l -v
```

看到发行版且 VERSION 为 2、STATE 为 Running 或 Stopped，即表示安装成功。进入 Linux 后可执行 `uname -a` 确认内核信息。

初始化：

```bash
sudo apt update && sudo apt upgrade -y
```

## 10. 网络、DNS 与代理

先分清两个配置文件：`.wslconfig` 在 Windows 用户目录（`%USERPROFILE%\.wslconfig`），管所有发行版；`/etc/wsl.conf` 在发行版内部，只管这一个发行版。**两者改完都要 `wsl --shutdown` 再重进才生效。**

### 10.1 默认 NAT 模式

WSL 2 有独立的虚拟网卡和 IP（每次重启可能变），和 Windows 之间是 NAT 关系。

- **Windows → WSL**：不用记 IP。WSL 2 自带 localhost 转发，在 Windows 浏览器里直接开 `http://localhost:8080` 就能访问 WSL 里的服务。兜底时可以查 IP：

  ```powershell
  wsl hostname -I
  ```

- **WSL → Windows**：用默认网关地址（也就是 Windows 主机）：

  ```bash
  ip route show default | awk '{print $3}'
  ```

- **局域网其他机器 → WSL**：NAT 模式下默认不通，要手动做端口转发（**管理员** PowerShell），并放行防火墙：

  ```powershell
  wsl hostname -I          # 先拿到 WSL IP，例如 172.20.10.5
  netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=172.20.10.5
  New-NetFirewallRule -DisplayName "WSL 8080" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow
  ```

  删除：`netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0`。WSL 的 IP 变了要重新加，很烦——所以能用镜像模式就用镜像模式。

### 10.2 镜像模式（推荐，Windows 11 22H2+ / WSL 2.0.0+）

在 `%USERPROFILE%\.wslconfig` 里写：

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
```

镜像模式下 WSL 和 Windows 共享网络接口：localhost 双向互通、IPv6 可用、VPN 兼容性更好，**不需要端口转发**，也能自动继承 Windows 的系统代理。

### 10.3 DNS

- 默认每次启动自动生成 `/etc/resolv.conf`，指向 Windows 的 DNS（NAT 模式下是虚拟网关地址）。
- 连公司 VPN 或内网域名解析不了时，按顺序试：
  1. `.wslconfig` 里加 `dnsTunneling=true`（见上）。
  2. 还不行就关掉自动生成，自己维护：

     ```ini
     [wsl2]
     generateResolvConf=false
     ```

     然后在发行版内自己写 `/etc/resolv.conf`，例如 `nameserver 10.0.0.2`。发行版重启后可能被覆盖，改完确认一下。

### 10.4 代理

- 只有「镜像模式 + `autoProxy=true`」会自动继承 Windows 的代理，NAT 模式不会。
- 手动配置（写进 `~/.bashrc` 或 `/etc/environment`），注意 **NAT 模式要用 Windows 主机 IP，镜像模式用 `127.0.0.1`**：

  ```bash
  export http_proxy=http://<主机IP>:7890
  export https_proxy=$http_proxy
  export no_proxy=localhost,127.0.0.1
  ```

- `sudo` 不会继承代理，需要 `sudo -E <命令>`；`apt` 单独配 `/etc/apt/apt.conf.d/95proxy`：

  ```
  Acquire::http::Proxy "http://<主机IP>:7890";
  Acquire::https::Proxy "http://<主机IP>:7890";
  ```

- 验证：`curl -I https://www.google.com`、`env | grep -i proxy`。

## 11. 禁止在 WSL 中调用 Windows 程序

WSL 默认开启 interop，并且把 Windows 的 `PATH` 追加到 Linux 的 `PATH` 后面。结果就是 Linux 里能直接跑 `notepad.exe`、`explorer.exe .`，`which code` 可能指向 Windows 的 exe，补全和脚本里也容易混进 .exe。

关掉它：编辑发行版内的 `/etc/wsl.conf`（需要 sudo，没有就新建）：

```ini
[interop]
enabled = false
appendWindowsPath = false
```

两项可以单独用：

| 配置 | 效果 |
| --- | --- |
| `appendWindowsPath = false` | 只清理 `PATH`，interop 仍可用：绝对路径 `/mnt/c/Windows/explorer.exe .` 还能调，`code .` 这类靠 Windows PATH 的命令会失效 |
| `enabled = false` | 彻底禁止执行 `.exe`，`explorer.exe` 也会失败 |

**副作用（选 `enabled = false` 前想清楚）**：

- 所有 `.exe` 调用失效，包括部分依赖 interop 的工具（Docker Desktop 集成、从 Linux 里启动 Windows GUI 程序等）。
- `code .` / `code-insiders .` 用不了；VS Code Remote-WSL 不受影响（它由 Windows 侧的 `wsl.exe` 拉起，走的是远程连接，不是 interop）。
- 改完必须 `wsl --shutdown` 再重进发行版才生效。

验证：

```bash
echo $PATH                     # 不应出现 /mnt/c/...
which notepad.exe              # 应该没有输出
cat /proc/sys/fs/binfmt_misc/WSLInterop   # enabled = false 时显示 disabled
```

**仅临时禁用**（重启发行版即恢复，适合跑一次干净的测试）：

```bash
sudo sh -c 'echo 0 > /proc/sys/fs/binfmt_misc/WSLInterop'
```
