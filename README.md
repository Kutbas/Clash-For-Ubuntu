# Clash-For-Ubuntu
一个适用于没有图形界面的Ubuntu系统的Clash部署方案

环境：Ubuntu Server 20.04 LTS 64bit

## 下载 Clash 的 Linux 版本

1. 本教程使用的版本为**clash-linux-amd64-v1.18.0**，地址：https://github.com/Kuingsmile/clash-core/releases 如果链接挂了可在本仓库下载。

2. 解压：

   ```bash
   gzip -d clash-linux-amd64-v1.18.0.gz
   ```

3. 对该文件赋予可执行权限：

   ```bash
   chmod +x clash-linux-amd64-v1.18.0
   ```

4. 更改文件名为：**clash**

5. 为方便进行全局访问，将文件移动至系统的**bin**目录下：

   ```bash
   sudo mv clash /usr/local/bin
   ```

6. 测试能否正常运行：

   ```bash
   clash -v
   ```

   ![image-20250408141524867](https://graphbed-1331926955.cos.ap-shanghai.myqcloud.com/GraphBed/202504081844676.png)

   如果能显示 clash 的版本信息，说明可以正常运行。

## 配置 .yaml 文件

当初次启动 clash 时，程序会生成一个 `~/.config/clash/config.yaml` 文件，我们节点的信息就可以配置在这个文件中。

如果此前已经在 Windows 系统上用 clash 订阅了机场，我们就可以直接把那边的配置信息复制到 Ubuntu 上的 `config.yaml` 文件中。

![image-20250408142547903](https://graphbed-1331926955.cos.ap-shanghai.myqcloud.com/GraphBed/202504081801118.png)

配置完成后，重新启动 clash，如果有如下提示信息，说明 clash 已经开始运行。

```txt
INFO[0001] RESTful API listening at: [::]:9090
```

## 开启代理

clash 正常运行后，我们要开启代理才能访问外网。

如果**只想在当前终端开启代理**，可执行如下操作：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=socks5://127.0.0.1:7891
```

通过下面的指令进行测试：

```bash
curl https://www.google.com
```

如果能够返回 html 内容，那么说明代理生效了。

如果要取消代理，执行：

```bash
unset http_proxy
unset https_proxy
unset all_proxy		
```

如果**想让代理对所有的终端生效**，那么我们需要在 `.bashrc` 进行配置。

```bash
sudo vim ~/.bashrc
```

在文件最后添加如下内容：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=socks5://127.0.0.1:7891
```

`wq!`保存，并执行：

```bash
source ~/.bashrc
```

## 在后台运行 Clash

如果想让 clash 在后台运行，可以创建 **Systemd 服务**。

1. 创建服务文件：

   ```bash
   sudo vim /etc/systemd/system/clash.service
   ```

2. 文件内容（假设 clash 路径为：`/home/ubuntu/Clash/clash`）：

   ```ini
   [Unit]
   Description=Clash Proxy Service
   After=network.target
   
   [Service]
   ExecStart=/home/ubuntu/Clash/clash
   Restart=always
   User=ubuntu
   WorkingDirectory=/home/ubuntu/Clash
   StandardOutput=file:/home/ubuntu/Clash/clash.log
   StandardError=file:/home/ubuntu/Clash/clash-error.log
   
   [Install]
   WantedBy=multi-user.target
   ```

3. 启用并启动服务：

   ```bash
   sudo systemctl daemon-reexec
   sudo systemctl daemon-reload
   sudo systemctl enable clash
   sudo systemctl start clash
   ```

4. 查看运行状态：

   ```bash
   sudo systemctl status clash
   ```

   ![image-20250408144552017](https://graphbed-1331926955.cos.ap-shanghai.myqcloud.com/GraphBed/202504081806178.png)

   当看到状态为”running“，说明 clash 服务已经在后台开始运行。

## 切换节点

在终端环境下，如果要切换节点，可按如下步骤进行：

1. 安装 JSON 处理工具 **jq**：

   ```bash
   sudo apt install jq
   ```

2. 获取当前节点列表：

   ```bash
   curl -s http://127.0.0.1:9090/proxies | jq
   ```

   执行命令后，会看到类似下面的输出：

   ```json
   {
     "proxies": {
       "节点选择": {
         "type": "Selector",
         "now": "香港01",
         "all": [
           "香港01",
           "日本01",
           "新加坡01"
         ]
       },
       "香港01": {
         "type": "Shadowsocks",
         "udp": true
       },
       ...
     }
   }
   ```

也可以看到当前节点。

![image-20250408210931109](https://graphbed-1331926955.cos.ap-shanghai.myqcloud.com/GraphBed/202504082135633.png)

其中，`Selector` 表示选择器，切换节点时，我们需要先确定主选择器，再确定要切换的节点。

3. 以我的配置文件为例，其中选择器名为 `🚀 节点选择`（空格变为`%20`），要切换到 `新加坡01` 节点，那么命令应该是：

   ```bash
   curl -X PUT http://127.0.0.1:9090/proxies/🚀%20节点选择 \
     -H "Content-Type: application/json" \
     -d '{"name":"新加坡01"}'curl -s http://127.0.0.1:9090/proxies/🚀%20节点选择 | jq
   ```

4. 查看节点是否切换成功：

   ```bash
   curl -s http://127.0.0.1:9090/proxies/🚀%20节点选择 | jq
   ```

   
## 基于 SSH 远程端口转发的服务器 HTTPS 访问解决方案

**1. 背景**

最近服务器遇到如下问题：

- 服务器 **无法正常访问 HTTPS 网站**

- 使用 `curl`、`git`、SDK 访问外部服务时，在 TLS 握手阶段报错：

  ```
  OpenSSL SSL_connect: SSL_ERROR_SYSCALL
  ```

- 即使配置了 HTTP / SOCKS5 代理：

  - TCP 连接正常
  - CONNECT 隧道正常
  - **TLS ClientHello 阶段连接被直接中断**

经过排查可知，这类问题**并非客户端配置错误或证书问题**，而是：

> **服务器出口网络或代理环境无法正常转发 TLS 流量**

但与此同时，注意到**本地 Windows 主机**具备：

- 正常的外网 HTTPS 访问能力
- 可用的本地代理（如 Clash、直连网络等）

因此，得到一个新的思路：

> **让服务器不直接访问外网 HTTPS，而是通过 SSH 隧道，将所有网络流量“回送”到本地 Windows 主机，再由本地主机完成真实的 HTTPS 访问。**

**2. 解决方案总体思路**

采用 **SSH 远程端口转发（Remote Port Forwarding）** 的方式，实现如下架构：

```
[Linux 服务器上的 curl / SDK]
        |
        |  SOCKS5（127.0.0.1:1080）
        v
[服务器本地 SSH 监听端口]
        |
        |  SSH 加密隧道
        v
[Windows 本地 SOCKS5 / 网络出口]
        |
        v
[Google / OpenAI 等 HTTPS 服务]
```

核心思想：

- **服务器只与本地主机进行 SSH 通信**
- **真正的 HTTPS / TLS 握手发生在 Windows 本地**
- 从而绕开服务器出口对 TLS 的限制

**3. 环境与前置条件**

**(1) 环境说明**

- 本地机器：Windows 10 / 11
- 服务器：Linux（Ubuntu / CentOS 等）
- 本地 ↔ 服务器：已配置 SSH 登录
- 本地网络：可正常访问 HTTPS 网站

**(2) 本地代理情况**

假设 Windows 本地已有可用的 SOCKS5 代理，例如：

```
127.0.0.1:7891
```

（无论该代理来自 Clash、v2ray 还是直连，均可）

**4. 实施步骤**

**(1) 在 Windows 本地建立 SSH 远程端口转发**

在 **Windows PowerShell / CMD** 中执行（注意：不是在服务器上执行）：

```bash
ssh -N -R 1080:127.0.0.1:7891 user@server_ip
```

参数说明：

| 参数                     | 含义                                      |
| ------------------------ | ----------------------------------------- |
| `-N`                     | 不执行远程命令，仅用于端口转发            |
| `-R 1080:127.0.0.1:7891` | 在服务器监听 1080 端口，并转发到本地 7891 |
| `user@server_ip`         | 服务器 SSH 登录信息                       |

执行后：

- SSH 会要求输入服务器密码
- 登录成功后终端**不会有任何输出**
- 窗口保持“卡住”状态是**正常现象**
- **不要关闭该窗口**

**(2) 在服务器上配置代理环境变量**

在服务器终端中执行：

```bash
export all_proxy=socks5h://127.0.0.1:1080
unset http_proxy https_proxy
```

说明：

- 使用 `socks5h`，确保 **DNS 解析也走隧道**
- 避免混用 HTTP / HTTPS 代理变量

**(3) 验证代理是否生效**

```bash
curl -v https://www.google.com
```

若能看到：

- TLS ServerHello
- HTTP 200
- 页面内容返回

效果：

![](https://graphbed-1331926955.cos.ap-shanghai.myqcloud.com/GraphBed/202603022141421.png)