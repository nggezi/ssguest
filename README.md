# ssguest - OpenWrt Shadowsocks + SOCKS5 管理

在 OpenWrt / ImmortalWrt / iStoreOS / KWRT 等泛 OpenWrt 系统上管理多个 Shadowsocks 节点，并可选开启一个带随机账号密码的 SOCKS5 服务。添加节点时自动生成随机密码、分配端口，并在防火墙里添加对应的 WAN 入站通信规则；删除节点时同步删除规则。

## 一键安装

在路由器 SSH 里执行（root）：

~~~sh
curl -fsSL -o /tmp/ssguest https://raw.githubusercontent.com/nggezi/ssguest/main/ssguest && sh /tmp/ssguest
~~~

脚本会安装 shadowsocks-libev-ss-server（已安装则跳过），把自身安装到 /usr/bin/ssguest，然后进入菜单。之后直接执行 ssguest 即可再次打开菜单。

国内访问 GitHub raw 慢时，可在地址前加镜像，例如：

~~~sh
curl -fsSL -o /tmp/ssguest https://ghfast.top/https://raw.githubusercontent.com/nggezi/ssguest/main/ssguest && sh /tmp/ssguest
~~~

## 菜单

~~~
 1) 添加 Shadowsocks 节点
 2) 删除 Shadowsocks 节点
 3) 查看 Shadowsocks 节点
 4) SOCKS5 管理
 5) 退出
~~~

添加节点时：

- 端口可留空自动分配，默认从 8388 开始找空闲端口；
- 密码随机生成 20 位字母数字；
- 加密方式默认 chacha20-ietf-poly1305；
- 同时写入防火墙通信规则，命名规则为 Allow-Guest 加节点号，例如节点 1 是 Allow-Guest1；
- 规则内容为源区域 wan、协议 tcp udp、目标端口为节点端口、动作 ACCEPT，兼容 fw3 和 fw4。

删除节点会停止对应进程、删除节点配置，并把同名防火墙规则一起删除。

关于防火墙：Allow-GuestN 是允许 WAN 入站到路由器本机的 input 规则，源区域选 wan，目标区域留空（或选“设备/输入”）。不要把它改成“WAN → LAN + 目标 IP 192.168.1.1”，那会生成 forward 规则，SS 服务在路由器本机上时不会生效。脚本在更新规则时会自动清掉 dest 和 dest_ip，防止这类误改。

## SOCKS5

菜单选 4 进入 SOCKS5 子菜单，可以开启/重建、查看、关闭删除。SOCKS5 使用随机用户名密码，默认用户名 guest，密码 20 位随机字母数字，端口从 10808 到 10820 自动分配，防火墙规则名为 Allow-GuestSocks（只放行 TCP）。

SOCKS5 服务端优先使用固件源里的 microsocks；如果没有，会自动检测 CPU 架构，从 GitHub 下载单文件静态 gost。检测到中国大陆网络时优先走 ghfast.top 等镜像，国外网络优先直连 GitHub。支持常见 amd64、arm64、armv7/armv6/armv5、mips/mipsle、mips64、loong64、riscv64 等架构，识别失败可用 SSG_GOST_ARCH 手动指定。

注意：gost 静态二进制约 43MB，下载压缩包约 15MB，需要路由器有足够剩余空间。小闪存设备建议先执行 opkg update && opkg install microsocks；脚本会自动优先使用它。

客户端地址示例：

~~~text
socks5://guest:随机密码@路由器公网IP:10808
~~~

需要远端 DNS 的客户端把 socks5:// 换成 socks5h://。microsocks 只转发 TCP，UDP 不适用。

SOCKS5 命令：

~~~sh
ssguest socks            # 打开 SOCKS5 子菜单
ssguest socks-add        # 开启/重建,端口自动分配
ssguest socks-add 10808  # 指定端口
ssguest socks-list       # 查看账号密码和状态
ssguest socks-del        # 关闭并删除
~~~

## 加密方式

默认使用 chacha20-ietf-poly1305。它是 AEAD 加密，主流 SS 客户端都支持；在没有 AES 硬件加速的 ARM/MIPS 软路由上通常比 AES-GCM 更稳。x86 软路由也可以换成 aes-256-gcm：

~~~sh
SSG_METHOD=aes-256-gcm ssguest add
~~~

已经创建的节点会保留创建时的加密方式。

## 命令行

~~~sh
ssguest                  # 菜单
ssguest add              # 添加节点,端口自动分配
ssguest add 9000         # 添加节点并指定端口
ssguest del 1            # 删除节点 1(会确认)
ssguest list             # 查看节点、密码和 ss:// 链接
ssguest host ss.example.com   # 固定链接里的域名/IP
ssguest socks            # SOCKS5 管理
~~~

如果路由器有公网 IP，list / 添加节点后会尝试自动探测公网 IPv4 生成 ss:// 链接。域名或固定 IP 可用 ssguest host 设置，保存在 /etc/ssguest/host。

## 文件位置

| 路径 | 说明 |
| --- | --- |
| /usr/bin/ssguest | 管理脚本 |
| /etc/init.d/ssguest | procd 服务，开机启动所有节点 |
| /etc/ssguest/nodes/节点号.json | ss-server 配置，权限 0600 |
| /etc/ssguest/nodes/节点号.meta | 节点端口、密码、加密方式等元数据 |
| /etc/ssguest/host | 可选的固定域名/IP |
| /etc/init.d/ssguest-socks | SOCKS5 的 procd 服务 |
| /etc/ssguest/socks.conf | SOCKS5 端口、用户名、密码、引擎 |
| /usr/bin/ssguest-socks-run | SOCKS5 启动包装脚本 |
| /usr/bin/ssguest-gost | 架构不匹配、没有 microsocks 时下载的静态 gost |

## 常用维护

~~~sh
/etc/init.d/ssguest status
/etc/init.d/ssguest restart
/etc/init.d/ssguest-socks status
/etc/init.d/ssguest-socks restart
logread | grep ss-server
~~~

如果菜单显示“服务: 运行中”但节点状态是“未监听”，先执行：

~~~sh
ss-server -h
~~~

能正常输出版本号才说明程序可以启动。若提示缺少 `libsodium.so.27` 之类的共享库，执行：

~~~sh
opkg update && opkg install libsodium
/etc/init.d/ssguest restart
~~~

新版脚本已经会在启动前做这个检查，并在缺依赖时自动尝试修复。

## 卸载

~~~sh
/etc/init.d/ssguest stop
/etc/init.d/ssguest disable
/etc/init.d/ssguest-socks stop
/etc/init.d/ssguest-socks disable
rm -f /etc/init.d/ssguest /etc/init.d/ssguest-socks
rm -f /usr/bin/ssguest /usr/bin/ssguest-socks-run /usr/bin/ssguest-gost
rm -rf /etc/ssguest
~~~

然后在 LuCI 的「网络 → 防火墙 → 通信规则」里删除残留的 Allow-Guest* 规则，或者用 UCI 删除。shadowsocks-libev-ss-server 软件包不会自动卸载，避免影响系统里其他 SS 用途。

## 说明

- 只支持路由器本体直接作为服务端、WAN 入站可访问的场景；如果路由器在 CGNAT 后面，需要自己做端口映射或内网穿透。
- 每个节点是一个独立的 ss-server 进程，由同一个 procd 服务管理，单个进程异常时 procd 会自动拉起。
- 如果修改了 SSG_FW_PREFIX，只影响之后新建的节点；旧节点的规则仍按旧前缀删除。
- SOCKS5 默认优先 microsocks（只支持 TCP）；没有时才下载 gost。可用 SSG_SOCKS_ENGINE=gost 强制使用 gost，或用 SSG_GH_MIRROR 指定下载镜像。
