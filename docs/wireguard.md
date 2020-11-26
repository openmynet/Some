# wireguard 部署文档

> 本文档依赖 docker 进行部署

## 前期准备工作

```bash
# 拉取wireguard镜像，有条件的可以自行构建docker镜像
sudo docker pull linuxserver/wireguard
```

## 1. 服务端

### 1.1 docker-compose 配置

```yaml
version: "3"

services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    ports:
      - 12340:12340/udp # 端口号要和wireguard配置文件中保持一致
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=0
      - PGID=0
      - TZ=Asia/Shanghai # 东8时区
    volumes:
      - /lib/modules:/lib/modules
      - /usr/src:/usr/src
      - ./wireguard/server.conf:/config/wg0.conf:ro # 配置文件映射
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

### 1.2 wireguard 服务端配置文件

> 重要：
>
> 1. 以下配置中`[Interface]/PostUp`以及`[Interface]/PostDown`,请配置好对应的网卡，否则将无法进行正常的通信
> 2. `wg0`： 对应的是由 wireguard 生成的虚拟网卡，在物理机中网卡名的配置文件的名称一致
> 3. `eth0`： 对应的是数据转发的主网卡，通常 docker 中默认是 eth0，linux 服务器中可能是 enp3s1 或者 enp1s0 等名称具体情况使用 `sudo ifconfig` 命令查看

```ini
[Interface]
  PrivateKey = 服务端私钥
  Address = 10.0.0.1/24
  PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
  PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
  ListenPort = 12340
  DNS = 8.8.8.8
  MTU = 1420

[Peer]
  # name = linux
  PublicKey = linux客户端公钥
  AllowedIPs = 10.0.0.2/32

[Peer]
  # name = windows
  PublicKey = windows客户端公钥
  AllowedIPs = 10.0.0.3/32

[Peer]
  # name = android
  PublicKey = Android客户端公钥
  AllowedIPS = 10.0.0.4/32
```

## 2. linux 客户端

### 2.1 docker-compose 配置

```yaml
version: "3"

services:
  wireguard_client:
    image: linuxserver/wireguard
    container_name: wireguard_client
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    volumes:
      - /lib/modules:/lib/modules
      - ./wireguard/linux_client.conf:/config/wg0.conf:ro # 配置文件映射
    network_mode: "host" # 主机模式方便访问内网
```

#### 2.2 wireguard linux 客户端配置文件

```ini
[Interface]
PrivateKey = linux客户端私钥
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = linux服务端公钥
AllowedIPs = 10.0.0.0/24 # 必须是有效的cidr，起始位是0，以保证兼容性
Endpoint = 1.1.1.1:12340 # 服务端IP:端口
PersistentKeepalive = 21

```

## 3. window 客户端

> 下载地址：有条件可以去官网下载
> 没有条件可以在此处下载，不保证安全性： <https://tlanyan.me/wireguard-clients/>

```ini
[Interface]
PrivateKey = window客户端私钥
Address = 10.0.0.3/24
DNS = 8.8.8.8

[Peer]
PublicKey = linux服务端公钥
AllowedIPs = 10.0.0.0/24 # 不要 0.0.0.0/0 否则会断网（win10系统）
# AllowedIPs = 0.0.0.0/1, 128.0.0.0/1 # 全局走vpn
Endpoint = 1.1.1.1:12340 # 服务端IP:端口
PersistentKeepalive = 21

```

## 4. android 客户端

> 下载地址： <https://f-droid.org/en/packages/com.wireguard.android/>

```ini
[Interface]
PrivateKey = android客户端私钥
Address = 10.0.0.4/24
DNS = 8.8.8.8

[Peer]
PublicKey = linux服务端公钥
AllowedIPs = 10.0.0.0/24 # 必须是有效的cidr，起始位是0而非1，因此10.0.0.1/24是错误的
Endpoint = 1.1.1.1:12340 # 服务端IP:端口
PersistentKeepalive = 21

```

## 5. 其他

1. 使用 `sudo docker ps` 查看容器运行情况
2. 使用 `sudo docker logs 容器ID` 查看 docker 容器运行日志
3. `[#] ip link delete dev wg0` 异常： 需要主机更新 linux 内核，注意不是 docker 容器本身
4. `sudo docker exec -it 容器ID /bin/bash` 进入容器
