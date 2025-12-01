# SSH-ZeroTier-ParaView-Client-Server

**pvserver 就是用来“画图”的那一端**：

* 它在 Linux 服务器上读 OpenFOAM 数据、做滤波/算场等
* 然后把结果通过网络发给你 Windows 上的 ParaView 客户端
* Windows 负责真正的“画图”和交互

所以你现在这一整套：`pvserver（服务器） + ParaView（本地）`
就是标准的 **Client–Server 远程后处理工作流**。


## 🧾 1. Daily Remote Visualization SOP


# Remote OpenFOAM Visualization via ZeroTier + SSH + ParaView

This document describes the daily workflow for visualizing large OpenFOAM cases
on a remote Ubuntu server using ParaView Client–Server mode.
```markdown
- Server OS: Ubuntu 22.04
- Server hostname: `sjtu-desktop`
- Server user: `sjtu`
- ZeroTier IP (server): `10.243.162.93`
- ParaView on server: 5.6.3 (OpenFOAM ThirdParty)
- ParaView on client (Windows): 5.6.2
- SSH tunnel & pvserver port: **11111**
```
You can adapt these values to your own setup.

---

## 0. Prerequisites (done once)

- ZeroTier network is created and both:
  - Windows laptop and
  - `sjtu-desktop` (Ubuntu server)
  are **joined** and **authorized**.
- From Windows you can ping the server:

```powershell
  ping 10.243.162.93
```
* OpenSSH client is available on Windows (`ssh` works in PowerShell).
* ParaView 5.6.2 (Windows, non-MPI) is installed, e.g.:

  ```text
  ParaView-5.6.2-Windows-msvc2015-64bit.exe
  ```

---

## 1. Start SSH tunnel (Windows)

Open **PowerShell** on Windows and run:

```powershell
ssh -L 11111:localhost:11111 sjtu@10.243.162.93
```

* Enter the password when prompted.
* **Do not close this window**.
  It is the SSH tunnel that forwards local port `11111` → server `11111`.

---

## 2. SSH login to the server (Windows)

Open a **second** PowerShell window and run:

```powershell
ssh sjtu@10.243.162.93
```

This shell will be used to control the server (run `pvserver`, check files, etc.).

Change to your OpenFOAM case directory, for example:

```bash
cd /media/sjtu/sdb/srs
```

If needed, create a `.foam` file:

```bash
touch srs.foam
```

---

## 3. Start `pvserver` on the server (Linux)

In the SSH shell (second PowerShell window, already logged into Linux) run:

```bash
pvserver --server-port=11111
```

Expected output:

```text
Waiting for client...
Connection URL: cs://sjtu-desktop:11111
```

Keep this terminal open.
`pvserver` is now waiting for a ParaView client to connect.

---

## 4. Start ParaView (Windows client)

On Windows, start **ParaView 5.6.2**:

* via Start Menu, or
* by running:

  ```text
  ...\ParaView 5.6.2-Windows-msvc2015-64bit\bin\paraview.exe
  ```

---

## 5. Configure and connect to the remote server

In ParaView (Windows):

1. Go to **File → Connect…**

2. Click **Add Server…**

3. Configure:

   * **Name:** `foam-remote` (any name is fine)
   * **Server Type:** `Client / Server`
   * **Host:** `localhost`
   * **Port:** `11111`
   * **Startup Type:** `Manual`

4. Save the configuration.

5. Select `foam-remote` in the list and click **Connect**.

If the connection is successful:

* On Linux (pvserver terminal) you will see:

  ```text
  Client connected
  ```

* In ParaView (Windows) you will see a green “Connected” indicator at the bottom.

---

## 6. Open the OpenFOAM case (remote filesystem)

In ParaView (Windows):

1. Go to **File → Open…**

2. In the file dialog, switch to **Remote file system** (not Local).

3. Browse to your case directory, e.g.:

   ```text
   /media/sjtu/sdb/srs
   ```

4. Select `srs.foam` (or your own `.foam` file).

5. Click **Open**, then **Apply** in the Properties panel.

You can now interactively visualize the large OpenFOAM case using the remote
server’s `pvserver` and your local ParaView client.

---

## 7. Clean shutdown (recommended order)

When you are done:

1. **Close ParaView** (Windows GUI).

2. On the Linux `pvserver` terminal, press:

   ```text
   Ctrl + C
   ```

   to stop `pvserver`.

3. On the Windows tunnel PowerShell window (the one with `ssh -L ...`), press:

   ```text
   Ctrl + C
   ```

   to close the SSH tunnel.

4. On the Windows SSH shell (logged into Linux), type:

   ```bash
   exit
   ```

   to end the SSH session.

ZeroTier can remain running in the background.
Next time, simply repeat sections **1–6**.

---

## 8. Troubleshooting notes

* **`Connection refused` when running `ssh sjtu@10.243.162.93`**
  → SSH service on the server may be stopped.
  Log in via local console or remote desktop and run:

  ```bash
  sudo systemctl restart ssh
  ```

* **`address already in use` when starting `pvserver`**
  → Some process is occupying port 11111. On Linux, run:

  ```bash
  sudo ss -tlnp | grep 11111
  ```

  Kill the offending process (e.g., old `pvserver` or `ssh -L`) and restart:

  ```bash
  sudo pkill pvserver
  sudo pkill ssh
  pvserver --server-port=11111
  ```

* **`Client/server version hash mismatch`**
  → ParaView client and `pvserver` versions are incompatible.
  Use ParaView 5.6.x on Windows to match server ParaView 5.6.3.

````

---

## 🖥 2. 一键启动脚本：自动开隧道 + SSH 登录

你现在每天要手动敲两条命令：

1. `ssh -L 11111:localhost:11111 sjtu@10.243.162.93`
2. `ssh sjtu@10.243.162.93`

我们可以弄一个 **Windows 批处理脚本 `.bat`**，双击就自动开两个 PowerShell 窗口：

- 一个跑隧道（`ssh -L`，窗口标题叫 “SSH tunnel”）  
- 一个跑普通 SSH 登录（标题叫 “SSH shell”）

你可以在仓库里建个 `scripts/` 目录，放一个 `start_remote_viz.bat`：

```bat
@echo off
REM ============================================================
REM  start_remote_viz.bat
REM  - Open SSH tunnel for ParaView Client–Server
REM  - Open an interactive SSH shell to the server
REM ============================================================

set SERVER_USER=sjtu
set SERVER_IP=10.243.162.93
set TUNNEL_PORT=11111

REM --- Start SSH tunnel in a new PowerShell window ---
start "SSH tunnel" powershell -NoExit ^
  ssh -L %TUNNEL_PORT%:localhost:%TUNNEL_PORT% %SERVER_USER%@%SERVER_IP%

REM --- Start normal SSH shell in another PowerShell window ---
start "SSH shell" powershell -NoExit ^
  ssh %SERVER_USER%@%SERVER_IP%
````

> 说明：
>
> * `start "窗口标题" powershell -NoExit ssh ...`
>   会打开一个新的 PowerShell 窗口并保持打开
> * 第一行窗口用于跑隧道，第二行窗口用于普通 ssh 登录

你可以在 README 里再加一个 “Helper scripts” 部分，例如：

````markdown
## Helper scripts

A helper batch script is provided in `scripts/start_remote_viz.bat`:

- Starts an SSH tunnel for ParaView Client–Server:
  - `local:11111 → server:11111`
- Opens an interactive SSH shell to the server.

Usage (Windows):

```text
Double-click `start_remote_viz.bat`.
- A window titled "SSH tunnel" will appear (keep it open).
- A window titled "SSH shell" will appear; use it to run `pvserver`:
  pvserver --server-port=11111
````


