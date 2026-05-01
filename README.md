# BugFix

- Fix 时间：2026-05-01 15:34:07 CST
- 简介：修复 Linux 同步脚本可能会匹配到 `Typora` 文件夹路径的问题，造成误判，已改为精确检测真实 Typora 进程。

---

# Typora Sync

本文件夹通过一个脚本被追踪和自动上传`github`，该脚本会每隔5s检测是否有`Typora`相关进程：

  - 开启`Typora`——触发一次`git pull`操作
  - 关闭`Typora`——触发一次`git push`操作

具体介绍如下。为了方便不同操作系统的用户，本文档提供了 **Linux** 和 **Windows** 两种平台的配置方法。

请注意，您首先需要确保在您的文档文件夹下存在一个已有的仓库，并且已经和github远端仓库链接，这是因为脚本的功能本质是执行`git`操作，请先完成本地创建与远程同步，确保 终端/命令行 中可以执行git操作后再使用

## 平台选择

<details>
<summary><strong>🐧 Linux 用户操作指南</strong></summary>

### 脚本代码

`typora-sync.sh`

```bash
#!/bin/bash

# --- 用户配置 ---
REPO_PATH="/home/fan0100/文档/MarkDown"
# ----------------

if [ ! -d "$REPO_PATH" ]; then
    echo "错误：仓库路径 '$REPO_PATH' 不存在。请修改脚本中的 REPO_PATH 变量。"
    exit 1
fi

cd "$REPO_PATH" || exit

echo "守护进程启动：正在监控 Typora 并自动同步。"
echo "仓库路径: $REPO_PATH"

is_typora_running=0
session_commit_count=0

while true; do
    # 检查 Typora 是否正在运行
    if pgrep -f "Typora" > /dev/null; then
        # Typora 刚刚启动
        if [ $is_typora_running -eq 0 ]; then
            echo "[$(date)] Typora 启动，正在拉取远程仓库..."
            # 先 stash 未提交的更改，然后 pull
            git stash push -m "typora-sync-stash-$(date +%s)"
            if git pull origin main --rebase; then
                git stash pop || git stash drop # 如果pop失败(无冲突)，则drop
                notify-send "Typora Sync" "拉取远程仓库完成"
                echo "[$(date)] 拉取完成。"
            else
                notify-send "Typora Sync 出错" "拉取失败！请手动处理 Git 冲突"
                echo "[$(date)] 错误：拉取失败！请手动解决冲突。"
                # 在拉取失败时退出，避免后续操作导致问题
                exit 1
            fi
            is_typora_running=1
            session_commit_count=0
        fi

        # Typora 运行中，创建常规提交
        # 检查是否有未暂存的更改
        git add -A
        if ! git diff --cached --quiet; then
            echo "[$(date)] 检测到更改，创建临时自动提交..."
            git commit -m "Auto-sync @ $(date +'%Y-%m-%d %H:%M:%S')"
            session_commit_count=$((session_commit_count + 1))
        fi

    else
        # Typora 刚刚关闭
        if [ $is_typora_running -eq 1 ]; then
            echo "[$(date)] Typora 关闭，准备同步最终更改..."
            
            git add -A
            # 捕获最后一次更改
            if ! git diff --cached --quiet; then
                 git commit -m "Auto-sync @ $(date +'%Y-%m-%d %H:%M:%S')"
                 session_commit_count=$((session_commit_count + 1))
            fi

            # 如果本会话有任何提交，则将它们打包成一个
            if [ $session_commit_count -gt 0 ]; then
                echo "[$(date)] 本次会话产生了 ${session_commit_count} 个临时提交，正在打包..."
                
                # 找到与远程分支的共同祖先，这是我们本次会话开始前的位置
                MERGE_BASE=$(git merge-base HEAD origin/main)
                
                # 软重置到这个基点，所有更改都将保留在暂存区
                git reset --soft "$MERGE_BASE"
                
                # 用所有暂存的更改创建一个全新的、干净的提交
                FINAL_COMMIT_MSG="Typora Sync Session: $(date +'%Y-%m-%d %H:%M')"
                git commit -m "$FINAL_COMMIT_MSG"
                
                echo "[$(date)] 打包完成。准备推送到远程仓库..."
                
                # 在推送前，做最后一次同步，以处理竞争条件
                echo "[$(date)] 正在进行最终同步检查..."
                if git pull origin main --rebase; then
                    echo "[$(date)] 开始推送..."
                    if git push origin main; then
                    	notify-send "Typora Sync" "推送成功"
                        echo "[$(date)] 推送成功！"
                    else
                    	notify-send "Typora Sync 出错" "推送失败！请检查网络或权限"
                        echo "[$(date)] 错误：推送失败！请检查网络或权限。"
                    fi
                else
					notify-send "Typora Sync 出错" "最终同步失败！您的修改已打包成一个本地提交，请手动解决冲突后推送"
                    echo "[$(date)] 错误：最终同步失败！您的修改已打包成一个本地提交，请手动解决冲突后推送。"
                fi
            else
            	notify-send "Typora Sync" "未检测到任何更改，无需同步"
                echo "[$(date)] 本次会话未检测到任何更改，无需同步。"
            fi
            
            is_typora_running=0
        fi
    fi

    # 减少轮询频率，降低资源占用
    sleep 5
done
```

### 自启动

下面将该脚本变成服务，并设置为自启动，运行在后台

#### 第一步：确认脚本信息

在开始之前，请确认您已经拥有以下两个关键信息：

1.  **脚本的绝对路径**：例如 `/home/fan0100/文档/typora-sync.sh`。
2.  **脚本需要运行的目录**（即您的Typora笔记仓库路径）：例如 `/home/fan0100/文档/MarkDown`。

#### 第二步：创建用户级服务单元文件

1.  创建用户服务所在的目录（如果它不存在的话）：

   ```bash
   mkdir -p ~/.config/systemd/user/
   ```

1. 打开终端，使用文本编辑器（这里用`nano`，您也可以用`vim`或`gedit`等）创建一个新的服务文件。这需要管理员权限，所以前面要加 `sudo`。

   ```bash
   sudo nano ~/.config/systemd/user/typora-sync.service
   ```

2. 将下面的模板内容**复制**并**粘贴**到打开的编辑器中。

   ```yaml
   [Unit]
   Description=Typora GitHub Sync Service
   # 确保此服务在图形会话启动后运行
   After=graphical-session.target
   
   [Service]
   # 替换成您的Typora笔记仓库的绝对路径
   WorkingDirectory=/home/fan0100/文档/MarkDown
   
   # 替换成您的脚本的绝对路径
   ExecStart=/home/fan0100/文档/typora-sync.sh
   
   Restart=always
   RestartSec=10
   
   [Install]
   # 将服务绑定到图形会话
   WantedBy=graphical-session.target
   ```

#### 第三步：理解并修改配置文件

现在，我们来逐行解释模板的含义，并告诉您如何修改以适应您自己的环境。

  - **`[Unit]` 部分**:
      - `Description`: 服务的描述，可以是任何您喜欢的名字，方便您在查看服务列表时辨认。
      - `After=graphical-session.target`: 这是一个非常重要的依赖关系。这个服务**会在图形化界面启动之后**才启动。因为我们的脚本依赖该界面弹出提示。
  - **`[Service]` 部分**:
      - `WorkingDirectory=/home/fan0100/文档/MarkDown`: **【请修改】** 指定脚本的默认工作目录。**请将其修改为您的Typora笔记仓库的绝对路径**。这样做可以确保脚本中的`git`命令在正确的目录下执行。
      - `ExecStart=/home/fan0100/文档/typora-sync.sh`: **【请修改】** 指定要执行的脚本的绝对路径。**请将其修改为您的`typora-sync.sh`脚本存放的绝对路径**。
      - `Restart=always`: 这是一个强大的功能。它表示如果脚本因为任何原因意外退出（比如崩溃了），`systemd`会一直尝试自动重启它。
      - `RestartSec=10`: 在重启前等待10秒，以防脚本因致命错误而陷入无限快速重启的循环。
  - **`[Install]` 部分**:
      - `WantedBy=graphical-session.target`: 这表示当系统会将服务绑定到图形会话。

**修改完成后，按 `Ctrl + X`，然后按 `Y`，最后按 `Enter` 保存并退出 `nano` 编辑器。**

#### 第四步：管理和运行服务

现在服务文件已经创建好了，我们需要用 `systemctl` 命令来控制它。

1.  **重新加载`systemd`守护进程**： 每次创建或修改了 `.service` 文件后，都需要运行此命令来让 `systemd` 重新读取配置。

    ```bash
    systemctl --user daemon-reload
    ```

2.  **设置服务开机自启**： 这个命令会创建一个符号链接，让服务在开机时自动启动。

    ```bash
    systemctl --user enable typora-sync.service
    ```

3.  **立即启动服务**： `enable`命令只负责设置开机自启，但不会立即运行它。我们需要手动启动第一次。

    ```bash
    systemctl --user start typora-sync.service
    ```

4.  **检查服务状态**： 这是最重要的调试命令，可以查看服务是否正在运行，以及最近的日志输出。

    ```bash
    systemctl --user status typora-sync.service
    ```

    如果一切正常，您会看到绿色的 `active (running)` 字样。

-----

#### 日常管理命令

现在，您已经拥有了一个真正的系统服务！以后您可以用以下命令来管理它：

  - **停止服务**: `sudo systemctl stop typora-sync.service`

  - **重启服务**: `sudo systemctl restart typora-sync.service`

  - **取消开机自启**: `sudo systemctl disable typora-sync.service`

  - **查看详细日志**:

    ```bash
    journalctl -u typora-sync.service
    ```

  - **实时查看日志**（像 `tail -f`）:

    ```bash
    journalctl -f -u typora-sync.service
    ```

</details>

<details>
<summary><strong>💻 Windows 用户操作指南</strong></summary>

### 脚本代码 (PowerShell)

将以下内容保存为 `typora-sync.ps1`。

```powershell
#Requires -Version 5.1

<#
.SYNOPSIS
    一个PowerShell守护进程脚本，用于在Typora运行时自动同步指定的Git仓库，并发送桌面通知。

.DESCRIPTION
    该脚本会监控Typora进程，并提供与Linux版本相同的功能。
    此版本使用Windows原生API发送通知，无需安装任何外部模块。

.NOTES
    作者: Gemini & User
    版本: 4.0 (原生通知版)
    平台: Windows PowerShell
#>

# --- 用户配置 ---
# 请将此路径修改为您要同步的Markdown仓库的绝对路径
$RepoPath = "C:\Users\xiaof\Documents\MarkDown"
# ----------------


# --- 函数定义 ---
# 封装一个发送Windows原生桌面通知的函数
function Send-Notification {
    param (
        [string]$Title,
        [string]$Message
    )
    
    # 加载发送通知所需的Windows核心程序集
    Add-Type -AssemblyName System.Windows.Forms
    
    # 检查当前PowerShell版本是否支持 Windows.UI.Notifications
    try {
        [Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] > $null
        [Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom.XmlDocument, ContentType = WindowsRuntime] > $null
    } catch {
        Write-Warning "无法加载Windows通知API。您的系统可能不支持此功能。"
        return
    }

    # 为我们的通知设置一个应用ID，这对于让通知正常工作是必需的
    $AppId = '{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe'
    $ToastNotifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AppId)

    # 创建通知内容的XML模板
    $ToastXml = New-Object Windows.Data.Xml.Dom.XmlDocument
    # 使用 ToastGeneric 模板，可以显示标题和内容
    $ToastXml.LoadXml(@"
<toast>
    <visual>
        <binding template='ToastGeneric'>
            <text>$($Title)</text>
            <text>$($Message)</text>
        </binding>
    </visual>
</toast>
"@)

    # 基于XML创建并显示通知
    try {
        $Toast = [Windows.UI.Notifications.ToastNotification]::new($ToastXml)
        $ToastNotifier.Show($Toast)
    } catch {
        # 捕获可能的错误，例如在某些服务器环境下
        Write-Warning "发送桌面通知失败。"
    }
}


# --- 脚本正文 ---

# 1. 验证并进入仓库目录
if (-not (Test-Path -Path $RepoPath -PathType Container)) {
    Send-Notification "Typora Sync 错误" "仓库路径不存在"
    Write-Host "错误：仓库路径 '$RepoPath' 不存在。" -ForegroundColor Red
    exit 1
}
cd $RepoPath

Write-Host "守护进程启动：正在监控 Typora 并自动同步..." -ForegroundColor Green
Write-Host "仓库路径: $RepoPath"
Send-Notification "Typora Sync" "守护进程已启动"

# 2. 初始化状态变量
$isTyporaRunning = $false
$sessionCommitCount = 0

# 3. 开始无限循环监控
while ($true) {
    $typoraProcess = Get-Process -Name "Typora" -ErrorAction SilentlyContinue

    if ($typoraProcess) {
        # Typora 刚刚启动
        if (-not $isTyporaRunning) {
            Write-Host "[$([datetime]::Now)] Typora 启动，正在拉取远程仓库..."
            git stash push -m "typora-sync-stash-$(Get-Date -Format 'yyyyMMddHHmmss')"
            git pull origin main --rebase
            if ($LASTEXITCODE -eq 0) {
                git stash pop
                Send-Notification "Typora Sync" "拉取远程仓库完成"
                Write-Host "[$([datetime]::Now)] 拉取完成。"
            } else {
                Send-Notification "Typora Sync 出错" "拉取失败！请手动处理 Git 冲突"
                Write-Host "[$([datetime]::Now)] 错误：拉取失败！请手动解决冲突。" -ForegroundColor Red
                git stash pop # 恢复更改以便用户解决
                exit 1
            }
            $isTyporaRunning = $true
            $sessionCommitCount = 0
        }

        # Typora 运行中，创建常规的临时提交
        if ((git status --porcelain).Length -gt 0) {
            Write-Host "[$([datetime]::Now)] 检测到更改，创建临时自动提交..."
            git add .
            git commit -m "Auto-sync @ $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"
            $sessionCommitCount++
        }

    } else {
        # Typora 刚刚关闭
        if ($isTyporaRunning) {
            Write-Host "[$([datetime]::Now)] Typora 关闭，准备同步最终更改..."
            
            if ((git status --porcelain).Length -gt 0) {
                 git add .
                 git commit -m "Auto-sync @ $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"
                 $sessionCommitCount++
            }

            if ($sessionCommitCount -gt 0) {
                Write-Host "[$([datetime]::Now)] 正在打包 $sessionCommitCount 个临时提交..."
                $MERGE_BASE = (git merge-base HEAD origin/main).Trim()
                git reset --soft $MERGE_BASE
                $finalCommitMessage = "Typora Sync Session: $(Get-Date -Format 'yyyy-MM-dd HH:mm')"
                git commit -m $finalCommitMessage
                
                Write-Host "[$([datetime]::Now)] 打包完成。准备推送到远程..."
                git pull origin main --rebase
                if ($LASTEXITCODE -eq 0) {
                    git push origin main
                    if ($LASTEXITCODE -eq 0) {
                        Send-Notification "Typora Sync" "推送成功！"
                        Write-Host "[$([datetime]::Now)] 推送成功！" -ForegroundColor Green
                    } else {
                        Send-Notification "Typora Sync 出错" "推送失败！请检查网络或权限"
                        Write-Host "[$([datetime]::Now)] 错误：推送失败！" -ForegroundColor Red
                    }
                } else {
                    Send-Notification "Typora Sync 出错" "最终同步失败！请手动解决冲突"
                    Write-Host "[$([datetime]::Now)] 错误：最终同步失败！" -ForegroundColor Yellow
                }
            } else {
                Send-Notification "Typora Sync" "本次会话无更改，无需同步"
                Write-Host "[$([datetime]::Now)] 本次会话未检测到任何更改，无需同步。"
            }
            $isTyporaRunning = $false
        }
    }

    # 轮询间隔
    Start-Sleep -Seconds 30
}


```

### 自启动

可以使用**NSSM**设置**Windows 服务**来实现开机自启和后台运行。

#### 第一步：准备工作

1. **允许脚本执行**: 以 **普通用户** 打开 PowerShell 窗口，运行以下命令并按 `Y` 确认：

   ```powershell
   Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
   ```

2. 使用`.\typora-sync.ps1`运行测试，如果出现乱码，记得保存为`Utf-8 BOM`格式

#### 第二步：创建服务

1. 访问官网 **[nssm.cc/download](https://nssm.cc/download)** 下载最新版的 ZIP 包。

2. 以 **管理员身份** 打开一个新的 PowerShell 窗口。

3. **安装新服务**: 运行 `nssm.exe install TyporaSync`。NSSM 的图形界面会弹出。

4. **配置 NSSM**:
     - **`Application` 选项卡**:
       - **Path**: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`
       - **Arguments**: `-NoProfile -ExecutionPolicy Bypass -File "C:\Users\xiaof\Documents\typora-sync.ps1"`
     - **`Log on` 选项卡 (防止1069登录失败和Git权限错误)**:
       - 勾选 **`This account`**。
       - **Account**: 输入您的 Windows 用户名 (例如 `xiaof`)。
       - **Password**: 输入您的 **Windows 登录密码**。**注意：如果您用 PIN 码登录，这里必须输入完整的微软账户密码，而不是 PIN 码。**
     - **`I/O` 选项卡**:
       - **Output (stdout)**: `C:\Users\xiaof\Documents\service-output.log`
       - **Error (stderr)**: `C:\Users\xiaof\Documents\service-error.log` *(这可以捕获脚本启动前的错误)*
     - 点击 **`Install service`** 按钮，等待安装成功的提示。


#### 第三步：启动与最终验证

1. **启动服务**:
   - 按 `Win + R`，输入 `services.msc` 并回车。
   - 在列表中找到您创建的 `TyporaSync` 服务。
   - 右键点击它 -> **属性** -> 将“启动类型”设置为“**自动 (Automatic)**” -> 点击“确定”。
   - 再次右键点击它 -> **启动**。
2. **最终验证**:
   - **服务状态**: 在 `services.msc` 中，确认 `TyporaSync` 的状态为“**正在运行**”。
   - **日志文件**: 打开 `service-error.log`、`service-output.log`，查看是否正常创建或写入
   - **端到端测试**: 再次启动、编辑、关闭 Typora，然后检查 log 文件和 GitHub 仓库，确认整个自动化流程已在后台无缝运行。

#### 日常管理

  - **服务管理**: 在电脑"服务"中打开，查找`TyporaSync`服务，可以启动、停止、配置服务
  - **查看运行情况**: 打开log文件即可查看运行状况

</details>

-----

## 安全机制与数据完整性

> **您的本地内容在绝大多数情况下是安全的，不会被自动覆盖。** 脚本的设计思路是将数据丢失的风险降到最低。

为了理解其安全性，我们分析一个最常见的场景：

> **场景**: 您在电脑A上修改了笔记，但关闭Typora时因网络中断，导致 `git push` 上传失败。
>
> 此时，您的修改已通过 `git commit` 保存在本地仓库中，只是尚未同步到远程。

当您下次打开Typora时，脚本会执行 `git pull origin main --rebase`，此命令包含两个核心步骤：

1.  **`git fetch`**: 从GitHub拉取最新的版本信息。此步骤**不会修改**您的任何本地文件。
2.  **`git rebase`**: 尝试将您本地尚未上传的修改，应用到刚拉取的最新版本“之上”。

根据远程仓库的状态，会发生以下两种情况：

-----

### 情况一：远程仓库无变化 (最常见)

这是指只有您自己在一台设备上工作，或者其他设备没有进行任何修改和同步。

  - **状态**: 您的本地仓库领先于远程仓库。
  - **`git pull` 行为**:
      - `git fetch` 发现远程没有任何新内容。
      - `git rebase` 发现您的本地修改已经是基于最新版本，因此什么也不做。
  - **结论**: ✅ **您的本地内容完好无损，完全不会被覆盖。**

-----

### 情况二：远程仓库有新内容 (多设备同步)

这是指在您上传失败的期间，您的另一台电脑B成功上传了新的内容。

  - **状态**:
      - **电脑A (本地)**: 包含您未上传的修改 (`A1`)。
      - **GitHub (远程)**: 包含了从电脑B上传的新修改 (`B1`)。
      - 此时，本地和远程的记录出现“分叉”。
  - **`git pull` 行为**:
      - `git fetch` 会将电脑B的修改 `B1` 下载到本地。
      - `git rebase` 会智能地尝试整合：
          - **无冲突**: 如果 `A1` 和 `B1` 修改的是不同文件或同一文件的不同区域，Git会自动将您的修改 `A1` 叠加到 `B1` 之后。最终结果是您的本地仓库**同时拥有了两方的修改**，完美合并，数据零丢失。
          - **有冲突**: 如果 `A1` 和 `B1` 修改了同一文件的同一行，会发生以下情况：
            1.  `git rebase` 命令会**立即停止**并报告冲突。
            2.  脚本会检测到Git命令执行失败，并打印警告：`[警告：'git pull' 失败，可能存在冲突。请手动解决！]`
            3.  最关键的是，**脚本不会再继续任何操作**，将决定权交还给您。您的文件不会被覆盖，而是被Git标记为“冲突状态”，等待您手动解决。

## 冲突处理指南

如果您在日志中看到了冲突警告，请不要担心，这说明Git需要您的帮助来决定如何合并修改。

您可以打开`vscode`或其他编辑器来辅助处理 git 冲突

或者您也可以仅通过命令行处理，运行 `git status`，它会清晰地告诉您哪个文件正处于冲突状态。

用任何文本编辑器（包括Typora）打开那个冲突的文件。您会看到类似 `<<<<<` `=====` `>>>>>` 的标记，Git用它来分隔来自不同版本的冲突内容。请根据您的需要，手动编辑文件，删除这些标记，并将内容修改为您想要的最终版本。

当您确认文件已修改无误后，执行以下命令：

```
# 告诉Git您已经解决了冲突
git add .

# 继续未完成的rebase过程
git rebase --continue
```

## 暂停脚本

- Linux:

  ```
  systemctl --user stop typora-sync.service
  ```

- **Windows**: 打开 **任务计划程序**，在 "任务计划程序库" 中找到你的任务，右键点击并选择 **"结束"**。
