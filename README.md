# SSH-ZeroTier-ParaView-Client-Server

## SOP
确定zerotier是正常连接的
```bash
ping 10.243.162.93
```
```bash
ssh -L 11111:localhost:11111 sjtu@10.243.162.93
```
在 Windows 另开一个 PowerShell（当前 SSH 这个窗口不要关）
```bash
ssh sjtu@10.243.162.93
```
在Linux中
```bash
pvserver --server-port=11111
```
在paraview里配置server文件\
然后在算例文件夹下
```bash
touch case_name.foam
```
就可以正常使用了！

关的时候\
先关 ParaView（GUI）

再停 pvserver（Linux，窗口 B）
   在运行 `pvserver` 的 ssh 窗口里按：

   ```text
   Ctrl + C
   ```

再关 SSH 隧道（Windows，窗口 A）
   在跑 `ssh -L ...` 的那个 PowerShell 窗口里按：

   ```text
   Ctrl + C
   ```

关 SSH 登录窗口（B）
   在 Linux shell 里输入：

   ```bash
   exit
   ```

ZeroTier 可以一直挂着，无所谓。
下次用时，从“建立 SSH 隧道”那一步重新走即可。



---

## 一、我们解决的是什么问题？什么时候适合用这套方案？

你的实际情况是：

* 服务器在实验室 **Linux 台式机（无 GPU）**
* 你在 **Windows 笔记本上** 想跑 OpenFOAM 大算例并看图
* 老师原方案：

  * 在 Linux 上跑算例
  * 用向日葵远控看图形界面
* 但：

  * 向日葵有延迟，刷新跟不上，大算例旋转/操作极其卡
  * 你经常在校外/异地，需要穿校园网、VPN、梯子，网络路径复杂

所以我们设计了一套更“专业”的方案：

> **Linux 负责算 + 数据处理，Windows 负责画图和交互**
> —— 用 **ZeroTier + SSH 隧道 + ParaView Client–Server**

这种配置适用于：

* ☑ 没 GPU、但要看大规模 OpenFOAM 结果
* ☑ 不想把动辄几十 GB 的结果文件拷到本地
* ☑ 不想一直靠远控桌面（向日葵/TeamViewer）
* ☑ 经常在校外，网络环境复杂（校园网 + VPN + 梯子）

不适用的情况：

* 只在本地小算例玩玩 → 本机装个 ParaView 直接打开就行
* 本地机器有很强的 GPU → 可以考虑把 case 拷回本地后处理

---

## 二、整体架构：这套东西是怎么“串起来”的？

你现在的最终架构可以一句话概括：

> **ZeroTier 做“虚拟局域网”，SSH 做“加密隧道”，pvserver 做“远端 post 处理核心”，Windows ParaView 做“前端 GUI”。**

具体链路：

1. **ZeroTier**

   * 让 Windows 和 Linux 虚拟出一个内网（10.x.x.x 那个 IP）
   * 避开校园网/VPN/梯子带来的各种路由问题
   * 你现在服务器的 ZeroTier IP 是：
     `10.243.162.93`

2. **SSH + 端口转发（隧道）**

   * 在 Windows 上开一条隧道：

     ```bash
     ssh -L 11111:localhost:11111 sjtu@10.243.162.93
     ```
   * 意思是：
     Windows 的 `localhost:11111` → 被转发到 Linux 的 `localhost:11111`

3. **Linux 上的 pvserver**

   * 在 Linux 上跑：

     ```bash
     pvserver --server-port=11111
     ```
   * 相当于在服务器上开了一个“可视化服务端”，等客户端连过来

4. **Windows 上的 ParaView Client（5.6.2）**

   * 你装了 ParaView 5.6.2（和 Linux 上的 5.6.3 同一个大版本）
   * 在 ParaView 里配置一个服务器：

     * Host: `localhost`
     * Port: `11111`
   * 通过 SSH 隧道去连 Linux 上的 pvserver

**这样一来：**

* 数据一直在 Linux 硬盘上，不搬家
* 计算也在 Linux 上做
* 你只在 Windows 上收“画图指令 + 渲染结果”，速度远比远程桌面稳和快

---

## 三、一次性配置要做的事（你已经搞完了）

这些是你已经完成、以后不用再重复的：

### 1. ZeroTier 配网

* 创建一个 ZeroTier 网络（你的是 `56374AC9A498D3FB`）

* Windows 加入该网络，后台授权（AUTH 打勾）

* Linux 安装 ZeroTier：

  ```bash
  curl -s https://install.zerotier.com | sudo bash
  sudo zerotier-cli join 56374AC9A498D3FB
  ```

* 在 ZeroTier 后台给 Linux 那一项也打勾 AUTH

* 最终在 Linux 上看到类似：

  ```bash
  sudo zerotier-cli listnetworks
  # ... OK PRIVATE ... 10.243.162.93/16
  ```

### 2. 安装 ParaView 版本匹配

* Linux：OpenFOAM 自带 ParaView 5.6.3（ThirdParty）
* Windows：单独安装 ParaView 5.6.2（non-MPI，msvc2015，64bit）

  * 文件名：`ParaView-5.6.2-Windows-msvc2015-64bit.exe`

**关键点：**
服务器端 pvserver 和客户端 ParaView 要在 **同一个大版本（5.6.x 系列）**，
否则就会报：

> Client/server version hash mismatch

---

## 四、日常使用的完整流程（每天从 0 到画出图）

### 0. 检查 ZeroTier 是否在线

在 Windows PowerShell：

```powershell
ping 10.243.162.93
```

能 ping 通（几 ms 延迟） → ZeroTier OK。

---

### 1. 在 Windows 上建立 SSH 隧道（窗口 A，**全程要开着**）

打开 PowerShell：

```powershell
ssh -L 11111:localhost:11111 sjtu@10.243.162.93
```

* 输入密码
* 登录进去后，这个窗口保持不动，**不要关**
* 它就是你 ParaView 的“高速通道”

> 🧠 记：**ssh -L 一定在 Windows 开，Linux 端绝对不要再跑这条命令**。
> 你之前的问题就是在 Linux 上也开了 `ssh -L`，把 11111 端口占了。

---

### 2. 在 Windows 上再开一个 SSH 登录（窗口 B）

再开一个 PowerShell 窗口：

```powershell
ssh sjtu@10.243.162.93
```

这是你的“操作窗口”，用来：

* cd 到算例目录
* 创建 `.foam` 文件
* 启动 `pvserver`

例如：

```bash
cd /media/sjtu/sdb/srs
touch srs.foam
```

---

### 3. 在 Linux 上启动 pvserver（在窗口 B，**全程要开着**）

在 SSH 登录的这个窗口里运行：

```bash
pvserver --server-port=11111
```

看到：

```text
Waiting for client...
Connection URL: cs://sjtu-desktop:11111
```

说明 pvserver 啥事没有，在等客户端连。
**这个窗口同样要全程开着**，直到你画完图。

---

### 4. 在 Windows 上打开 ParaView 5.6.2（GUI）

通过开始菜单或 paraview.exe 打开：

> 重点：**一定是 5.6.2，不是你本地那个 5.13.3**

---

### 5. 在 ParaView 中配置并连接服务器

1. 菜单：**File → Connect…**

2. Add Server：

   * Name: 随便，比如 `foam-remote`
   * Server Type: Client/Server
   * Host: `localhost`
   * Port: `11111`
   * Startup Type: `Manual`（因为你已经在 Linux 手动启动了 pvserver）

3. Save 之后，在列表选中 `foam-remote` → 点击 Connect

**成功标志：**

* Linux 端 pvserver 的窗口里多一行：

  ```text
  Client connected
  ```
* Windows ParaView 右下角出现绿色连接图标

---

### 6. 打开远程 OpenFOAM case

在 ParaView 中：

1. **File → Open…**
2. 在顶部切换为 **Remote File System**（不是 Local）
3. 浏览到你的 case 目录，例如：

   ```text
   /media/sjtu/sdb/srs
   ```
4. 选择 `srs.foam`（或你自己的 `.foam` 文件）
5. 点 **Open** → 左侧点 **Apply**

此时你看到的网格/流场，是服务器读数据后的结果，通过 SSH 隧道传到你 Windows 渲染的。

---

## 五、用完之后如何“优雅正确地关掉一切”？

顺序建议如下（从内到外）：

1. **先关 ParaView（GUI）**
   → Windows 上直接叉掉窗口

2. **再停 pvserver（Linux，窗口 B）**
   在运行 `pvserver` 的 ssh 窗口里按：

   ```text
   Ctrl + C
   ```

3. **再关 SSH 隧道（Windows，窗口 A）**
   在跑 `ssh -L ...` 的那个 PowerShell 窗口里按：

   ```text
   Ctrl + C
   ```

4. **关 SSH 登录窗口（B）**
   在 Linux shell 里输入：

   ```bash
   exit
   ```

ZeroTier 可以一直挂着，无所谓。
下次用时，从“建立 SSH 隧道”那一步重新走即可。

---

## 六、常见坑 & 关键细节（你已经踩过的那些）

### 1. `Connection timed out` vs `Connection refused`

* **timeout** → 通常是网络不通（ZeroTier 掉线 / VPN 乱路由）
* **refused** → 通常是 Linux 上 sshd 挂了（你之前用 `pkill ssh` 把它干掉了）

解决 refused：

* 用向日葵/本地登录 Linux，执行：

  ```bash
  sudo systemctl restart ssh
  ```

---

### 2. `address already in use`（pvserver 启动时）

原因：11111 端口已经被占用了，常见两种情况：

* 之前的 pvserver 没关干净
* 你在 Linux 上误跑了 `ssh -L 11111:...` 自己占着端口

排查 & 解决：

```bash
sudo ss -tlnp | grep 11111
# 找到占用这个端口的进程 PID，比如 ssh pid=19690
sudo kill 19690
```

然后再跑：

```bash
pvserver --server-port=11111
```

---

### 3. `Client/server version hash mismatch`

原因：你的 Windows ParaView 和 Linux pvserver **版本不一致**，比如：

* Linux：ParaView 5.6.3
* Windows：ParaView 5.13.3

解决：Windows 上安装 **5.6.x 系列**（你现在是 5.6.2），只用它来连远程服务器。

---

### 4. 哪些窗口“必须全程开着”？

日常工作时，**至少要保持这几个一直存在**：

1. **Windows：SSH 隧道窗口（ssh -L 11111:localhost:11111 ...）**

   * 关闭 → 隧道断 → ParaView 断线

2. **Linux：pvserver 窗口**

   * 关闭/打断 → 可视化服务端没了 → ParaView 会断线

3. **（可选）Windows：SSH 登录窗口**

   * 不一定要一直开着，但一般都会留着方便敲命令
   * 就算关了也不影响已经运行中的 pvserver（除非你是在同一个会话里起的，没用 tmux）

---

## 七、什么时候优先考虑这种配置？

总结一下“这套东西值不值得折腾”的判断标准：

* ✅ 你的算例 **太大**，本地拉不动；
* ✅ 服务器是固定的 Linux 主机，存算例的硬盘也在那；
* ✅ 没 GPU，远程桌面渲染卡、人也难受；
* ✅ 经常在各种网络环境（宿舍、寝室、校外）远程连服务器；
* ✅ 希望有**长期稳定的科研工作流**，而不是临时凑合。

对于你现在 OpenFOAM + 飞鱼 / cavity / 两相流等题目，这套方案基本是“正解”。

---

如果你愿意，我还能帮你再加一个简短版（比如 10 行的“超速上手版”，贴在 README 最上面当 cheat sheet），或者帮你写一个 `tmux` 使用小节，让你即使 SSH 断了、pvserver 或求解器也不会被杀。你要的话我可以直接写好给你。
