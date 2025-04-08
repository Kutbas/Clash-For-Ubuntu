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

   ![image-20250408141524867](https://raw.githubusercontent.com/Kutbas/GraphBed/main/Typora/202504082135037.png)

   如果能显示 clash 的版本信息，说明可以正常运行。

## 配置 .yaml 文件

当初次启动 clash 时，程序会生成一个 `~/.config/clash/config.yaml` 文件，我们节点的信息就可以配置在这个文件中。

如果此前已经在 Windows 系统上用 clash 订阅了机场，我们就可以直接把那边的配置信息复制到 Ubuntu 上的 `config.yaml` 文件中。

![image-20250408142547903](https://raw.githubusercontent.com/Kutbas/GraphBed/main/Typora/202504082136900.png)

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

   ![image-20250408144552017](https://raw.githubusercontent.com/Kutbas/GraphBed/main/Typora/202504082136694.png)

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

![image-20250408210931109](https://raw.githubusercontent.com/Kutbas/GraphBed/main/Typora/202504082135633.png)

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

   
