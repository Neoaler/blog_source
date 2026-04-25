---
abbrlink: ''
categories:
- - nas
date: '2026-04-25T19:42:42.328827+08:00'
tags:
- nas
- taliscale
- mihomo
title: truenas 安装 taliscale 和 mihomo
updated: '2026-04-25T19:42:42.568+08:00'
---
## taliscale

由于我的 truenas 版本是 25.10.3，在 b 站看到别人的教程里直接就是在应用商店有 taliscale 安装包，但是我的应用里啥也没有

![image](https://img2.eval.moe/image-20260424174658-ij3e97z.png)

然后去查了一下为什么我的 truenas 应用商店里没有 taliscale，其实也不仅仅是没有 taliscale，b站那个视频里它的版本是 22 的，有六七百个应用，我的这个 25 版本的才只有 376 个应用，查了一下，原来如此，原来是架构给换了，很多东西就不支持了

之前听说过 truenas 就是不能更新，一更新就指定会出什么毛病，可能也是因为经常换架构啥的

![image](https://img2.eval.moe/image-20260424174537-vyjsbt8.png)

所以我只能用 yaml 安装的方式装我的 taliscale 了——其实说白了也就是 docker compose 的方式安装，填完点击保存，等一会儿就装好了

![image](https://img2.eval.moe/image-20260424175523-3qd2j3d.png)

```
services:
  tailscale:
    cap_add:
      - net_admin
    container_name: tailscale
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - >-
        TS_AUTHKEY=<ts-key>
      - TS_HOSTNAME=<hostname>
      - TS_ROUTES=192.168.1.0/24
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_EXTRA_ARGS=--accept-routes=false
    image: tailscale/tailscale:stable
    network_mode: host
    privileged: True
    restart: unless-stopped
    volumes:
      - ./tailscale:/var/lib/tailscale
```

## mihomo

mihomo 的安装就有点复杂，主要是有个叫 geoip 的文件得手动下载，并上传上去，和上面一样，依旧通过 yaml 安装

```
services:
  clash: # Clash 内核
    container_name: clash-meta
    image: metacubex/mihomo:v1.18.4  # 最新版用 metacubex/mihomo:Alpha
    restart: always
    pid: host
    ipc: host
    network_mode: host
    cap_add:
      - ALL
    security_opt:
      - apparmor=unconfined
    volumes:
      - ./mihomo:/root/.config/mihomo
      - /dev/net/tun:/dev/net/tun
      # 共享host的时间环境
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro

  metacubexd: # Clash WebUI
    container_name: metacubexd
    image: ghcr.1ms.run/metacubex/metacubexd
    restart: always
    network_mode: bridge
    ports:
      - '127.0.0.1:28002:80'
    volumes:
      - ./metacubexd:/config/caddy
      # 共享host的时间环境
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
```

docker 启动后，发现起不来，查看 clash 的日志显示

![image](https://img2.eval.moe/image-20260425182250-q8ga31s.png)

这时候就要手动下载：[Country.mmdb](https://cdn.jsdelivr.net/gh/Dreamacro/maxmind-geoip@release/Country.mmdb "Country.mmdb")，然后放到 clash 的挂载目录里

![image](https://img2.eval.moe/image-20260425182426-e68lx2s.png)

重新启动 docker 后就正常了
