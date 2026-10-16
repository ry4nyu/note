# WSL 安装指南

WSL（Windows Subsystem for Linux）让你在 Windows 上直接运行 Linux 环境。下面以 WSL 2 为准。

## 1. 环境要求

- Windows 10 版本 2004 及以上（内部版本 19041+），或 Windows 11。
- 64 位系统。
- 在 BIOS/UEFI 中开启虚拟化（Intel 为 VT-x，AMD 为 SVM）。
  - 任务管理器 → 性能 → CPU，若显示“虚拟化: 已启用”即可。

## 2. 一键安装（推荐）

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

安装指定版本，例如 Debian：

```powershell
wsl --install -d Debian
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

## 6. 手动安装（`wsl --install` 不可用时）

Windows 版本较旧时 `wsl --install` 可能不支持，需要手动启用功能，以管理员身份执行：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

重启电脑后，下载并安装 WSL 内核更新包（搜索“WSL2 Linux kernel update package for x64”），再执行第 4 节的设置命令，然后从 Microsoft Store 安装 Ubuntu。

## 7. 常见问题

**1. 提示“未启用虚拟机平台”或“请在 BIOS 中启用虚拟化”**
进入 BIOS 开启 Intel VT-x / AMD SVM，保存后重启。

**2. 提示 `WSL 2 requires an update to its kernel component`**
执行 `wsl --update`，然后重启 WSL：`wsl --shutdown`。

**3. `wsl --install` 没有反应或报错 `无效的命令行选项`**
Windows 版本过低，参考第 6 节手动安装，或先更新 Windows。

**4. 安装时卡在 0%**
检查网络，或稍等几分钟；可以关闭后重新执行 `wsl --install`。

**5. 想换安装位置**
WSL 的数据默认在 `%LOCALAPPDATA%\Packages\` 下。可用 `wsl --export <名称> <tar路径>` 导出，再用 `wsl --import <名称> <安装目录> <tar路径>` 导入到其他盘。

## 8. 验证安装

在 PowerShell 中执行：

```powershell
wsl -l -v
```

看到发行版且 VERSION 为 2、STATE 为 Running 或 Stopped，即表示安装成功。进入 Linux 后可执行 `uname -a` 确认内核信息。
