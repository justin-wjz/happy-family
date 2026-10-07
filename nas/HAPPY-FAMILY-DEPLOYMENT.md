# 家庭服务器部署、运维与 LLM 接手说明

> **状态基线：2026-10-07，Asia/Shanghai（UTC+8）。**
> 当天晚间复核了主机、容器、systemd、挂载、主要配置和软件版本。
> 对象：`xlucky@192.168.1.182` 上的 Nextcloud、HTTPS、dynv6 DDNS 和 Trojan 客户端。
> 建议存放位置：`justin-wjz/happy-family/docs/HOME-SERVER-DEPLOYMENT.md`。
>
> 这是**已运行系统的交接文档，不是一键重装脚本**。其中版本、IP、证书期限和状态是快照，未来操作前必须重新核实。文档不包含密码、Token、私钥、数据库口令或会话 Cookie；仅凭本文件不能恢复这些凭据。
>
> 目标 GitHub 仓库为公开仓库。本文保留了部署所需的主机名、网络地址、用户名和目录结构，上传前请确认接受公开这些基础设施信息。不要附加服务器上的原始配置、日志或备份。

## 1. 给接手 LLM 的首要约束

1. **唯一推荐的用户访问入口是 `https://gavin-nas.dns.army:8443`。** 用户已经明确要求关闭 HTTP。不要恢复主机 80 端口发布或路由器的 80/8081 映射，也不要为了证书验证开放 HTTP。
2. **原始 USB 数据盘仍连接旧主机 `192.168.1.114`。** 新主机通过加密 SMB 挂载使用它。不是已经把 USB 插到了 `.182`。不要关闭旧主机、切断其网络或重新启动其旧 Nextcloud 容器。
3. 只有新主机上的 Nextcloud 应写入这套生产数据。不要同时启动旧、新两套应用，不要重新初始化现有数据目录。
4. 用户此前明确选择**不复制、不备份 USB 上的用户数据**。数据库、应用和配置备份保存在内置磁盘；这并不等于完整数据备份。执行大规模数据复制、删除、转换或 USB 搬迁前必须获得新的授权。
5. MariaDB、应用、缓存、临时文件和日志尽量放内置磁盘。不要把数据库移到 USB；也不要承诺 USB 一定能休眠，索引、预览、同步和后台任务仍可能唤醒它。
6. Trojan **手动开关、开机默认关闭**。当前 DDNS 已兼容 VPN 开启，不能再按旧说明认定“VPN 开启时 DDNS 必须暂停”。调试需临时开 VPN 时，先确认授权并恢复用户原有状态。
7. 证书机构继续使用 **Let’s Encrypt**，DNS 使用 **dynv6**。不要未经授权切换 CA、购买域名、开通付费服务或修改 Azure 资源。
8. 不要输出 `.env`、完整 `config.php`、完整 sing-box 配置、ACME 账户配置或凭据文件。即使某个文件名看起来只是“迁移配置”，也可能包含数据库密码。
9. 不要为了排错运行 `docker compose down -v`、删除卷、递归 `chown/chmod` 用户数据，或执行历史迁移脚本。迁移已完成至 Nextcloud 35，不应重跑。
10. 本文中的读操作通常安全；重启、重建容器、改 DNS、换密钥、升级和恢复备份都属于变更操作，需要先评估影响。不要把“阅读交接文档”视为变更授权。

## 2. 快速状态表

| 项目 | 当前配置 / 最后确认状态 |
| --- | --- |
| 新主机 | `xlucky@192.168.1.182`，主机名 `xlucky` |
| 系统 | Ubuntu 24.04.4 LTS，x86_64 |
| 网络接口 / 网关 | `wlp1s0` / `192.168.1.1` |
| Docker / Compose | Docker 29.5.3 / Docker Compose v5.1.4 |
| Nextcloud | 显示版本 35.0.1，内部版本 35.0.1.1；installed=true、maintenance=false、needsDbUpgrade=false |
| 公网访问 | `https://gavin-nas.dns.army:8443` |
| Memories | `https://gavin-nas.dns.army:8443/apps/memories/` |
| Photos | `https://gavin-nas.dns.army:8443/apps/photos/` |
| 家庭公网 IPv4 | 最后确认 `120.245.103.202`，**动态地址，不是配置常量** |
| dynv6 zone | `gavin-nas.dns.army`，zone ID `6040363`，目前只配置 IPv4 |
| HTTP | 公网和 LAN 的 Nextcloud HTTP 均已关闭 |
| TLS | Caddy 2.11.7 + acme.sh 3.1.6 + Let’s Encrypt DNS-01 |
| DDNS | systemd 定时器，每约 5 分钟执行，已启用开机调度 |
| 证书检查 | 每天 04:00 起加最多 2 小时随机延迟，支持补执行 |
| Trojan | sing-box 1.14.2；手动开关，inactive、boot disabled |
| 旧存储主机 | `firefly@192.168.1.114`，继续提供 SMB 和 USB 存储 |
| 上游 Trojan 主机 | `azureuser@4.194.215.55`；不是 Nextcloud 的 HTTPS 反代或 DDNS 地址 |

`.182` 的局域网地址必须稳定：现有 Docker 端口绑定和路由器 NAT 都依赖它。此前记录的 Wi-Fi MAC 为 `0c:dd:24:1a:48:84`。**尚未确认完成 DHCP 静态租约**；不要在不了解 DHCP/静态地址现状时盲目改路由器。

## 3. 总体架构与请求路径

```text
用户浏览器 / Nextcloud 客户端
  |
  | DNS: gavin-nas.dns.army -> 家庭当前公网 IPv4
  | HTTPS TCP 8443
  v
家庭路由器 192.168.1.1
  | NAT: 公网 8443 -> 192.168.1.182:443
  v
192.168.1.182:443
  | Docker 发布到 Caddy 容器 :8443
  v
Caddy (nextcloud-https-caddy-1, 172.19.0.10)
  | 验证过的正式证书；TLS 在这里终止
  | Docker 内部 HTTP
  v
Nextcloud Apache/PHP (nextcloud-app-1)
  |-- MariaDB 10.11       [内部磁盘]
  |-- Redis              [Docker 内部网络]
  |-- cron               [相同应用和数据挂载]
  |-- go-vod             [内部网络，原始媒体只读，转码缓存内置磁盘]
  |
  v
/mnt/nextcloud-usb-data
  | SMB 3.1.1 + seal（加密）
  v
192.168.1.114 上的原始 USB 数据
```

### 3.1 路由器和端口的最终状态

路由器型号：Huawei OptiXstar V271-20。互联网 WAN 项目为 `2_INTERNET_R_VID_10`。

| 外部端口 | 内部目标 | 当前状态 |
| --- | --- | --- |
| TCP 80 | 原为 `.182:80` | **映射已删除** |
| TCP 8081 | 原为 `.182:80` | **映射已删除** |
| TCP 443 | `.182:443` | 映射保留，但此前外网连通测试未通过 |
| TCP 8443 | `.182:443` | 已通过外网 HTTPS 访问 |

新主机还直接发布 `.182:8443 -> Caddy:8443`，供 LAN 使用同一 HTTPS 域名和端口时使用。

公网 443/80 以前不通，即使存在映射和临时监听也如此；究竟是 ISP 过滤还是路由器保留端口，尚未最终确认。**不能因为保留了 443 映射，就省略访问 URL 中的 `:8443`。**

关闭 HTTP 的具体操作已经完成：

- 从 `/srv/nextcloud/compose.yaml` 的 `app` 中删除 `ports: ["192.168.1.182:80:80"]`。
- 只重建 `app` 容器；没有重建数据库或移动数据。
- 从路由器删除 80 和 8081 两条映射，重新读取页面确认只剩 443/8443。
- 主机不再监听 Nextcloud 的 80 端口；旧 `http://gavin-nas.qicp.vip:8081` 不再可用。

这里“关闭 HTTP”指关闭**用户可访问的明文入口**。Caddy 到 Apache、go-vod 等 Docker 内部 HTTP 仍然存在，没有发布到 LAN/公网；不要直接关闭容器内 Apache 的 80 端口。路由器自身的 LAN 管理网页不属于此次 Nextcloud HTTP 关闭范围。

### 3.2 Docker 网络

| 网络 | 范围 | 用途 |
| --- | --- | --- |
| `nextcloud-frontend` | `172.19.0.0/16`，gateway `.1`，internal=false | app、cron、Caddy |
| `nextcloud-backend` | `172.18.0.0/16`，gateway `.1`，internal=true | app、cron、数据库、Redis、go-vod |

Caddy 固定地址为 **`172.19.0.10`**，Nextcloud 信任代理配置依赖它。app 最近为 `172.19.0.2`，但不应把 app 的动态地址当配置契约；Caddy 上游使用 `nextcloud-app-1:80`。

HTTPS 是独立 Compose 项目，使用已存在的 `nextcloud-frontend` 外部网络。主应用的 systemd 单元只读取明确指定的主 Compose 文件，**不会自动应用任意另存的 override 文件**。

## 4. 数据存储：最容易误判的部分

### 4.1 当前 USB/SMB 关系

| 位置 | 含义 |
| --- | --- |
| 旧机 `/media/firefly/nas` | 原始 USB 的挂载点，历史确认 UUID `c484abec-7a27-4b05-8a9a-d72d0e4c5778` |
| `//192.168.1.114/nextcloud-migration` | 原有 Nextcloud 数据的 SMB share |
| 新机 `/mnt/nextcloud-usb-data` | 生产数据挂载，rw、SMB 3.1.1、seal |
| app/cron `/var/www/html/data` | 绑定生产挂载，保持迁移前容器数据路径 |
| go-vod `/var/www/html/data` | 相同数据，只读绑定 |
| `/home/xlucky/nextcloud-readonly` | 给普通用户浏览用的独立只读 SMB 挂载 |

生产挂载使用 uid/gid 33（www-data）；此前配置的文件/目录模式分别为 0660/0770，并有 hard、nosuid、nodev、noexec 等限制。挂载凭据仅保存在 root 私有文件：

```text
/srv/nextcloud-migration/secrets/smb.credentials
```

`nextcloud-data.service` 调用 `/srv/nextcloud-migration/mount-data.sh`，`nextcloud.service` 启动前再次调用该脚本并检查 `.ncdata`。

容器入口 `/srv/nextcloud/nextcloud-launch` 会检查 `/var/www/html/data/.ncdata` 中存在完整的一行：

```text
# Nextcloud data directory
```

现代 `.ncdata` 还可能有其他内容，不要要求整个文件等于这一行。挂载缺失时应拒绝启动，而不是把空目录初始化为新的 Nextcloud 数据目录。

**依赖不等于全自动故障恢复**：systemd 的 `BindsTo` 能跟随相关单元状态；SMB 连接失效却未改变单元状态时，不能假定它会自动识别全部存储故障。

### 4.2 内置磁盘上的实际目录

| 路径 | 用途 |
| --- | --- |
| `/srv/nextcloud/html` | 运行中的 Nextcloud 应用、config、custom_apps 等 |
| **`/srv/nextcloud/db-10.11`** | 当前 MariaDB 10.11 实际数据目录 |
| `/srv/nextcloud/logs` | Nextcloud/audit 日志，容器内 `/var/log/nextcloud` |
| `/srv/nextcloud/memories-cache` | go-vod 转码缓存，容器内 `/cache` |
| `/srv/nextcloud-migration` | 迁移记录、历史快照、存储保护脚本、数据库/SMB secret |
| `/srv/nextcloud-https` | HTTPS、DDNS、ACME、证书、相关备份 |
| Docker 本地卷 `/data` | Redis 当前使用本地匿名卷；不要误删 |

旧交接文档可能写 `/srv/nextcloud/db`。**当前数据库绑定是 `db-10.11`**，不要把旧目录当现用数据库恢复或删除。

appdata/预览等仍可能在数据目录中；“数据库在内置盘”不代表 USB 没有后台访问。

### 4.3 只读视图与账号权限

只读视图的单元为：

```text
/etc/systemd/system/home-xlucky-nextcloud\x2dreadonly.mount
```

它使用 uid/gid 1000、file_mode=0400、dir_mode=0500，挂载本身为 ro。

后来用户明确授权把 `xlucky` 加入 `www-data` 组。因此：

- `/home/xlucky/nextcloud-readonly` 仍只读。
- `xlucky` 账号**并非全局只读账号**，它也可能直接写入生产数据路径。
- 尽量通过 Nextcloud 管理文件；不要在宿主机直接改原始照片、版本和回收站，再靠全盘扫描补救。

## 5. Nextcloud 容器、镜像与启动依赖

主配置：

```text
/srv/nextcloud/compose.yaml
/srv/nextcloud/.env                  # 私密，不得公开
/srv/nextcloud/nextcloud-launch
```

| Compose service / 容器 | 镜像 | 用途 |
| --- | --- | --- |
| `app` / `nextcloud-app-1` | `nextcloud-local:35.0.1-video-20261006` | Apache/PHP，Nextcloud 35.0.1 |
| `cron` / `nextcloud-cron-1` | 同 app | `/cron.sh` 后台任务 |
| `db` / `nextcloud-db-1` | `mariadb:10.11`，有 digest 固定 | 数据库 |
| `redis` / `nextcloud-redis-1` | `redis:6.2-alpine`，有 digest 固定 | 缓存/锁相关服务 |
| `go-vod` / `nextcloud-go-vod-1` | `nextcloud-local/go-vod:0.4.2-nc35` | Memories 视频转码 |

固定的上游镜像：

```text
mariadb:10.11@sha256:dc9dd57be7f46ad6f350c598a331d64f295899c4abac3ec5652eff831fc38a6c
redis:6.2-alpine@sha256:b362b5dff9d14d961dc3352db4776aba6a8c53ca2661d7c74ad2c121f82cdaea
```

自定义视频镜像的 Dockerfile：

```text
/srv/nextcloud/images/video/Dockerfile
/srv/nextcloud/images/video/Dockerfile.transcoder
```

它们不是官方仓库可以直接拉取的同名镜像。重建新主机前应在私有备份中保存构建上下文或导出相应镜像；只有文档中的标签不足以恢复这些镜像。

主服务使用 `restart: "on-failure:5"`。app 等待 db、redis 健康，cron/go-vod 依赖 app 启动。app/cron 通过保护入口脚本启动，不要绕过它。

### 5.1 systemd 生命周期

```text
网络可用
  └─ nextcloud-data.service（确认并挂载原存储）
       └─ nextcloud.service（主 Compose）
            └─ nextcloud-https.service（独立 HTTPS Compose）
```

关键行为：

- `nextcloud.service`：Requires Docker 和数据单元；BindsTo 数据单元；PartOf Docker。
- `nextcloud-https.service`：Requires Docker 和 Nextcloud；PartOf Docker/Nextcloud。
- 这些是 `Type=oneshot`、`RemainAfterExit=yes`，正常显示 `active (exited)`；长期工作的实际进程在容器内。
- 启用的 systemd 单元负责开机启动；不能仅凭容器的 restart policy 推断完整开机行为。
- 停止/重启 Nextcloud 可能连带影响 HTTPS，不应把它当成无感操作。

主应用的 Compose 命令必须使用完整参数：

```bash
sudo docker compose \
  --env-file /srv/nextcloud/.env \
  -f /srv/nextcloud/compose.yaml \
  ps
```

读取配置结构时，优先 `config --quiet` 做语法检查。完整 `docker compose config` 可能展开 secret 环境变量，**不要把输出贴到聊天或提交到 GitHub**。

## 6. Nextcloud 配置优先级与 HTTPS 代理

运行中配置目录：

```text
/srv/nextcloud/html/config/
```

必须同时考虑：

- `config.php`
- 镜像/应用提供的 `*.config.php`
- `reverse-proxy.config.php`
- 历史迁移文件 `zz-migration.config.php`
- 此次新增的 `zzzz-https.config.php`

`zz-migration.config.php` **包含敏感配置**，不得因为要检查一个 URL 就整体打印它。

曾出现 `occ config:system:set overwrite.cli.url` 已成功写入主 `config.php`，但实际读取仍是旧 HTTP URL 的问题：后加载的迁移配置覆盖了它。最终新增只含下列内容的 `zzzz-https.config.php`，没有改写整个迁移文件：

```php
<?php
$CONFIG = [
    'overwrite.cli.url' => 'https://gavin-nas.dns.army:8443',
];
```

文件属主 www-data:www-data、权限 0640；已分别在 app 和 cron 容器用 `occ` 验证有效值。

### 6.1 当前关键有效设置

```text
trusted_domains 包含 gavin-nas.dns.army
trusted_proxies = ["172.19.0.10"]
forwarded_for_headers = ["HTTP_X_FORWARDED_FOR"]
overwritehost = gavin-nas.dns.army:8443
overwriteprotocol = https
overwritecondaddr = ^172\.19\.0\.10$
overwrite.cli.url = https://gavin-nas.dns.army:8443
```

历史 trusted_domains 中还保留旧域名、IP。允许某个 Host 不等于发布了相应网络入口；当前 HTTP 已由 Docker/NAT 层关闭。

条件覆盖仅对真正的 Caddy 地址生效，避免强行覆盖 Docker 内部请求。不要把 `trusted_proxies` 放宽为所有来源或整个互联网。

Apache 当前配置了 `RemoteIPHeader X-Real-IP`，且信任若干私网范围。Caddy 因而必须删除客户端伪造的 `X-Real-IP`；同时删除 `Forwarded` 和 `X-Forwarded-Port`，由 Caddy 正常生成受控的 X-Forwarded 信息。

### 6.2 历史保留策略

```text
versions_retention_obligation = disabled
trashbin_retention_obligation = disabled
activity_expire_days = 10000
```

前两项在这里表示关闭自动过期清理策略，不是禁用文件版本/回收站功能。不要为了释放空间擅自改变它们。

## 7. Photos、Memories 与视频转码

| 项目 | 版本 / 状态 |
| --- | --- |
| Photos | 8.0.0 |
| Memories | 9.0.1 |
| Preview Generator | 5.15.0 |
| FFmpeg | 7.1.5（容器包版本 `7.1.5-0+deb13u1`） |
| go-vod | 本地镜像 `0.4.2-nc35`，当前健康 |
| 硬件加速 | VAAPI=false、NVENC=false；当前主要是 CPU 转码 |

关键设置：

```text
memories.vod.external = true
memories.vod.connect = go-vod:47788
memories.vod.path = /var/www/html/custom_apps/memories/bin-ext/go-vod-amd64
```

`vod.path` 不表示应在 app 里再手工启动一个服务；当前实际使用独立 go-vod 容器。

重要页面区别：

- `/apps/photos/`：原 Photos 应用。
- `/apps/photos/memories`：Photos 的回顾页面，**不是独立 Memories 应用**。
- `/apps/memories/`：独立 Memories，可使用现有转码链路。

Windows 浏览器曾对 HEVC MOV/MP4 出现“有声音、无画面”；原始文件未损坏，也没有被替换。Memories 通过 HLS/H.264 转码实现播放，Gavin 的 timelinePath 配为 `/Photos`。

2026-10-07：

- 已通过 HTTPS 完整播放一个此前有问题的约 7.6 秒 MOV：230 解码帧、0 掉帧、无播放器错误。
- go-vod 对原始数据是只读，缓存写到内置盘。
- 全量 Gavin Photos 索引的历史临时单元为 `nextcloud-memories-index-gavin-20261007.service`。
- 后续查看日志确认其成功退出，但日志注明 **2 个失败文件被跳过，可用 `--retry` 重试**；尚未逐个处理这两个文件。
- 临时单元消失、`LoadState=not-found` 不代表任务没运行；应结合 journal 判读。

不要因页面无缩略图就盲目重复全量索引；它会读取大量媒体、唤醒 USB。重试失败文件应先检查日志和 `occ memories:index --help`。

## 8. Caddy HTTPS 部署

目录与主要文件：

```text
/srv/nextcloud-https/
  compose.yaml
  Caddyfile
  bin/caddy
  certs/fullchain.pem
  certs/key.pem
  data/
  config/
  reload.sh
/etc/systemd/system/nextcloud-https.service
```

Caddy 2.11.7 为官方发布二进制，下载时使用官方 SHA512 校验清单验证。运行在本地已有的 Alpine 基础镜像中：

```text
alpine@sha256:a2d49ea686c2adfe3c992e47dc3b5e7fa6e6b5055609400dc2acaeb241c829f4
```

这里**没有安装 Caddy dynv6 DNS 插件**。先前自定义下载接口不支持该模块，因此最终选择标准 Caddy 加独立 acme.sh。

### 8.1 Caddyfile

```caddyfile
{
    auto_https off
    admin 127.0.0.1:2019
    servers {
        protocols h1 h2
    }
}

https://gavin-nas.dns.army:8443 {
    tls /certs/fullchain.pem /certs/key.pem
    header Strict-Transport-Security "max-age=15552000"
    redir /.well-known/carddav /remote.php/dav/ 301
    redir /.well-known/caldav /remote.php/dav/ 301
    reverse_proxy nextcloud-app-1:80 {
        header_up -X-Real-IP
        header_up -Forwarded
        header_up -X-Forwarded-Port
    }
}
```

`auto_https off` 只关闭 **Caddy 自动申请证书/自动重定向机制**，并不关闭上面显式配置的 TLS。证书由 acme.sh 管理。

### 8.2 容器运行约束

- UID/GID `10001:10001`，只读 root filesystem。
- `cap_drop: ALL`、`no-new-privileges:true`。
- 二进制、Caddyfile、证书目录只读挂载。
- data/config 为专用可写目录，`/tmp` 使用受限 tmpfs。
- 管理 API 只监听容器内 `127.0.0.1:2019`，没有宿主机端口发布。
- 健康检查访问内部管理 API；它只能证明 Caddy 进程正常，不能单独证明 Nextcloud/存储/公网可用。
- 日志使用 json-file，单个日志上限 10 MiB、最多 3 个文件。
- 仅 TCP、HTTP/1.1 和 HTTP/2；没有部署 HTTP/3 UDP 转发。

## 9. Let’s Encrypt 证书与自动续期

### 9.1 为什么采用 DNS-01

家庭公网 80/443 不可依赖，而且已按用户要求移除了 HTTP 入口。DNS-01 通过 dynv6 API 添加 `_acme-challenge` TXT 验证域名控制权，不要求对公网开放 80/443。

固定证书身份：`gavin-nas.dns.army`。证书不绑定某个公网 IP，DDNS 更新地址不需要重新签发证书。

最近正式证书：

```text
签发完成：2026-10-07 17:48（北京时间）
Issuer：Let's Encrypt / YE2
SAN：gavin-nas.dns.army
NotAfter：2027-01-05 08:49:26 UTC
```

未来读取 live certificate 为准。没有使用自签或 staging 证书作为正式入口。

### 9.2 文件与命令角色

| 路径 | 作用 |
| --- | --- |
| `/srv/nextcloud-https/acme/` | acme.sh 3.1.6，官方固定版本解包安装 |
| `acme-state/` | 正式 ACME 账户、域名状态和内部证书文件；含敏感信息 |
| `acme-state-staging/` | 独立测试账户/证书状态，不可用于正式服务 |
| `certificate.sh` | `issue-staging`、`issue`、`renew`、`install`、`install-with-reload` |
| `certs/fullchain.pem`、`certs/key.pem` | 通过 `--install-cert` 部署的稳定路径 |
| `reload.sh` | 检查证书链、主机名、公私钥匹配，调整权限，强制重载 Caddy |
| `secrets/dynv6-token` | DNS API 凭据；DDNS 与 ACME 共用 |
| `logs/production-issue.log` | 首次自动签发的日志，不应未经检查公开 |

证书密钥和部署证书权限为 0600，属主是 Caddy 使用的数字 UID/GID 10001。宿主机上没有同名账号时 `stat` 显示 UNKNOWN 不代表属主丢失。

初次签发使用 EC P-256、`dns_dynv6` hook、`--dnssleep 90` 和明确的 Let’s Encrypt endpoint。90 秒等待不是 DNS 全网传播的保证。

### 9.3 正常续期

```ini
# /etc/systemd/system/nextcloud-cert-renew.timer
[Unit]
Description=Daily automatic Nextcloud certificate renewal

[Timer]
OnCalendar=*-*-* 04:00:00
RandomizedDelaySec=2h
Persistent=true
Unit=nextcloud-cert-renew.service

[Install]
WantedBy=timers.target
```

对应 service 以 root 运行 `certificate.sh renew`，超时 900 秒。acme.sh 根据证书状态和 CA 的 ARI 信息决定是否实际续期；每天运行不等于每天重新签发。

2026-10-07 的手动正常检查成功，因未到期而跳过；当时计划续期时间是 `2026-12-06T23:01:04Z`，这是快照而非承诺。

### 9.4 已结束的首次部署重试

曾遇到 Let’s Encrypt 对共享注册域 `dns.army` 的 50 张/168 小时额度限制。更换本地网络或开 Trojan 无法解除这类域名额度。

相关历史文件/单元仍保留：

```text
provision.sh
issue-not-before
provision-complete
provision.lock
nextcloud-https-provision.service
nextcloud-https-provision.timer
```

首次正式部署成功后，`nextcloud-https-provision.timer` 已自动停止并禁用。**不要把这个历史引导脚本作为日常续期任务重新启用**。它不是当前所有后续变更的完整重建脚本，尤其不能代替 HTTPS-only 配置和最后的配置覆盖文件。

## 10. dynv6 DDNS 的完整运行机制

### 域名与管理入口

- **DDNS 域名：`gavin-nas.dns.army`**，zone ID：`6040363`。
- **dynv6 官方接入说明页：<https://dynv6.com/zones/6040363/instructions>**。需要使用有权管理该 zone 的 dynv6 账号登录。
- DNS 记录管理页：<https://dynv6.com/zones/6040363/records>。
- 用户访问 Nextcloud 的地址：<https://gavin-nas.dns.army:8443>。上面的 dynv6 管理页面不是 Nextcloud 入口。

官方接入说明页用于查阅可用的更新方式；这台机器实际采用本文记录的 systemd + `ddns.py` + REST API 方案。不要把页面上可能包含 Token 的个性化更新 URL、命令或截图复制到公开仓库，也不要直接用通用示例覆盖现有 VPN 直连保护。

### 10.1 运行位置和调度

它运行在 `.182` **宿主机上**，不是路由器插件、不是 Docker 容器、不是需要常驻 SSH 的终端程序。

```text
nextcloud-ddns.timer
  -> nextcloud-ddns.service
     -> /usr/bin/python3 /srv/nextcloud-https/ddns.py
        -> 检查路由和分流策略
        -> HTTPS 获取两个一致的公网 IPv4
        -> GET 当前 dynv6 zone
        -> 有变化才 PATCH ipv4address
        -> GET 验证更新结果
```

路由器仍有其独立的 ORAY/花生壳 DDNS，但 **`gavin-nas.dns.army` 不依赖 `gavin-nas.qicp.vip`**，也不是通过 HTTP 跳转或 CNAME 指向它。不要给 DNS 记录填写 `http://`、路径或端口。

### 10.2 当前 IP 查询来源

按顺序尝试：

```text
https://ip.3322.net/
https://checkip.amazonaws.com/
https://ipinfo.io/ip
```

这些服务返回请求方的公网出口 IP。Trojan 分流保证这些特定请求从家庭网络直连。

算法细节：

- 使用 Python 标准库 HTTPS，验证证书与主机名，只查询 IPv4，不采用环境中的 HTTP proxy。
- 只允许预定义站点；当前还允许 API 目标 `dynv6.com`。
- 至少两个独立来源成功且一致，才继续；如果已经观察到冲突，不用“多数票”掩盖它。
- 前两个一致时即可停止，第三个主要是备用；并非每次必须查询三个。
- 拒绝私网/非公网 IPv4，也显式拒绝上游 Trojan 的已知出口 `4.194.215.55`。
- 请求失败会记录错误；不足两个可靠来源则退出失败，不写 DNS。
- 检查允许的路由接口；VPN 过渡/失败状态拒绝更新。
- VPN 活跃时验证预期的专用直连/DNS策略存在；不再简单地一律暂停。

### 10.3 dynv6 API

```text
Zone ID：6040363
Zone name：gavin-nas.dns.army
GET/PATCH https://dynv6.com/api/v2/zones/6040363
认证：HTTP Authorization 请求头采用 Bearer 方案，凭据从 root 私有文件读取，不写入命令行
更新内容：{"ipv4address": "<检测到的家庭公网 IPv4>"}
```

调用前检查返回的 zone ID 和名字匹配。地址未变则不 PATCH；PATCH 后再 GET 确认。IPv6 尚未部署，不会借此更新 AAAA。

### 10.4 实际 systemd 定义

```ini
# /etc/systemd/system/nextcloud-ddns.service
[Unit]
Description=Update the Nextcloud hostname with the home broadband IPv4 address
Wants=network-online.target
After=network-online.target
ConditionPathExists=/srv/nextcloud-https/secrets/dynv6-token

[Service]
Type=oneshot
UMask=0077
ExecStart=/usr/bin/python3 /srv/nextcloud-https/ddns.py
TimeoutStartSec=120
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=strict
CapabilityBoundingSet=
```

```ini
# /etc/systemd/system/nextcloud-ddns.timer
[Unit]
Description=Check the home broadband address every five minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
RandomizedDelaySec=30s
Unit=nextcloud-ddns.service

[Install]
WantedBy=timers.target
```

没有显式 User 时是 root 系统服务；CapabilityBoundingSet 为空不等于使用普通用户，root 仍按文件权限读取私有配置。

这是一次性任务。service 执行成功后 `inactive (dead)` 正常；timer 应是 `active (waiting)`，且 `enabled`。

公网 IP 改变后，检测周期加 DNS 缓存传播会造成短暂不可达。Linux 主机关机/断网时不会更新，但此时 Nextcloud 本身也不在线。

### 10.5 文件清单

```text
/srv/nextcloud-https/ddns.py
/srv/nextcloud-https/vpn_policy.py
/srv/nextcloud-https/check-ddns.py
/srv/nextcloud-https/configure-ddns-routing.py
/srv/nextcloud-https/verify-ddns-vpn.sh
/srv/nextcloud-https/store-dynv6-token.py
/srv/nextcloud-https/secrets/dynv6-token
```

- `ddns.py`：生产更新程序。
- `vpn_policy.py`：预期域名/直连/DNS策略及校验，**本身不是 sing-box 配置文件**。
- `check-ddns.py`：模拟测试，覆盖错误地址、来源冲突、错误 zone、PATCH/验证及 VPN 策略，不实际改 DNS。
- `configure-ddns-routing.py`：本次受控配置变更工具，会检查所提供的旧配置 SHA256、备份、运行 sing-box check 后原子替换。它不是可以盲目重复运行的通用安装器。
- `verify-ddns-vpn.sh`：会真的开启 VPN 的验证脚本，有恢复 trap 和独立延时关闭保障；**不是只读健康检查**，需要新的执行授权。
- `store-dynv6-token.py`：隐藏输入凭据，验证只授权预期 zone，写 root 0600；已有目标文件时拒绝覆盖。

## 11. Trojan / sing-box 客户端

### 11.1 用户命令与文件

```bash
trojan-vpn on
trojan-vpn off
trojan-vpn status
trojan-vpn logs
```

`on/off` 会在需要时调用 sudo。`on` 启动服务并检查 `trojan0` 的 DNS 就绪；若未就绪会停止服务恢复普通联网。命令不改变 boot enable 状态。

```text
/usr/local/bin/trojan-vpn
/usr/local/bin/sing-box
/etc/trojan-vpn/config.json            # 含密码；绝不公开
/etc/systemd/system/trojan-vpn.service
/var/lib/trojan-vpn/
```

客户端连接现有 Azure Trojan 服务：

```text
服务器 IPv4：4.194.215.55
服务器端口：443
关联域名：gavin-happyfamily.southeastasia.cloudapp.azure.com
密码：仅保存在私有运行配置中，本文故意不包含
```

保留现有 TLS 校验及 server_name 设置，不要为了连接成功开启 insecure/跳过证书校验。此次只修改客户端分流，未修改 Azure 上游服务。

Trojan 本质上是代理协议；这里由 sing-box 的 TUN/路由/DNS 集成提供接近系统 VPN 的使用方式。

### 11.2 原有路由骨架

- TUN 名称 `trojan0`，tag `tun-in`。
- TUN 地址：`172.28.255.1/30`、`fdfe:dcba:9876::1/126`。
- auto_route、auto_redirect、strict_route 开启，dns_mode 为 hijack。
- LAN、loopback、link-local 和 Azure 上游地址等有排除规则，避免 LAN 失联/代理回环。
- 默认 outbound：`trojan-out`。
- 普通 DNS：`tunnel-dns`，通过 Trojan 访问 `1.1.1.1` 的 DoH，TLS server_name 为 `cloudflare-dns.com`，IPv4-only。
- service 使用 `Restart=on-failure`，但 **没有 enable 开机启动**。

### 11.3 为 DDNS 新增的精确分流

当前精确域名集合：

```text
ip.3322.net
checkip.amazonaws.com
ipinfo.io
dynv6.com
```

新增 outbound：

{
  "type": "direct",
  "tag": "ddns-direct",
  "bind_interface": "wlp1s0",
  "domain_resolver": "ddns-dns"
}

新增 DNS server：

```json
{
  "type": "https",
  "tag": "ddns-dns",
  "server": "223.5.5.5",
  "server_port": 443,
  "path": "/dns-query",
  "tls": {
    "enabled": true,
    "server_name": "dns.alidns.com"
  },
  "detour": "ddns-direct"
}
```

这些域名的 DNS 规则置于优先位置，指向 `ddns-dns`。路由开头依次是：

```json
[
  {"port": 53, "action": "hijack-dns"},
  {
    "network": "tcp",
    "port": 443,
    "action": "sniff",
    "sniffer": ["tls"],
    "timeout": "1s"
  },
  {
    "domain": ["ip.3322.net", "checkip.amazonaws.com", "ipinfo.io", "dynv6.com"],
    "action": "route",
    "outbound": "ddns-direct"
  }
]
```

之后保留原有私网直连和默认 Trojan。TLS sniff 用来在透明 TUN 流量中识别域名，并不解密 HTTPS 内容。

以上是**配置片段，不是完整可运行的 config.json**。直接覆盖现有配置会丢失密码、TLS、TUN 和其余路由设置。

仅使用 `curl --interface wlp1s0` 曾不足以绕过 sing-box 的 nftables 重定向，因此最终使用 sing-box 内部的专用 direct outbound，而不是只在调用方绑定网卡。

### 11.4 已验证行为与维护边界

2026-10-07 18:45–18:46，经用户批准临时开启 VPN：

| 检查 | 结果 |
| --- | --- |
| 普通 `https://icanhazip.com` 请求 | 返回 Azure 出口 `4.194.215.55` |
| 三个 DDNS 白名单 IP 查询 | 全部返回家庭出口 `120.245.103.202` |
| `ddns.py --check-only`，VPN 开启 | 成功，vpn_active=true |
| 实际 `nextcloud-ddns.service`，VPN 开启 | 成功；IP 未变，无需写 A 记录 |
| Google `generate_204` | HTTP 204 |
| VPN 关闭后的 DDNS | 成功，仍是家庭出口 |
| 最后状态 | VPN inactive、boot disabled；DDNS timer 启用 |

先前用 `ipv4.icanhazip.com`/`api.ipify.org` 作为直连来源不够稳定，出现超时/重置，已从当前 DDNS 白名单移除。不要把旧列表重新覆盖回来。

VPN 活跃时的安全检查读取磁盘上的预期策略，不是万能的运行态证明。修改 sing-box 文件后，要先 `sing-box check`，再在批准的维护窗口重启活跃的服务并验证。只改文件、不重载，可能导致磁盘和运行态不一致。

如果更改 Wi-Fi 接口、上游 VPN 地址、域名列表或 sing-box 规则顺序，需要同时检查 `vpn_policy.py` 和 `ddns.py`，不能只改其中一个。

## 12. 日常维护：建议直接使用的命令

下列为 Linux 主机内执行的命令；从其他电脑先连接：

```bash
ssh xlucky@192.168.1.182
```

### 12.1 只读巡检

```bash
systemctl show nextcloud.service nextcloud-data.service nextcloud-https.service \
  -p Id -p ActiveState -p SubState -p UnitFileState

systemctl list-timers --all nextcloud-ddns.timer nextcloud-cert-renew.timer
systemctl show nextcloud-ddns.service -p Result -p ExecMainStatus
trojan-vpn status

sudo docker compose --env-file /srv/nextcloud/.env \
  -f /srv/nextcloud/compose.yaml ps
sudo docker compose -f /srv/nextcloud-https/compose.yaml ps

findmnt /mnt/nextcloud-usb-data
sudo docker exec -u www-data nextcloud-app-1 php occ status

curl --fail --show-error --silent \
  https://gavin-nas.dns.army:8443/status.php

sudo openssl x509 -in /srv/nextcloud-https/certs/fullchain.pem \
  -noout -subject -issuer -dates -ext subjectAltName
```

不要使用 `curl -k` 证明 HTTPS 正常。主机域名 DNS 故障时，可诊断性地绕过 DNS，仍验证 TLS 主机名：

```bash
curl --fail --show-error --silent \
  --resolve gavin-nas.dns.army:8443:192.168.1.182 \
  https://gavin-nas.dns.army:8443/status.php
```

这只能验证本地 TLS/应用，不证明公网 DNS/NAT 正常。

### 12.2 日志

```bash
sudo journalctl -u nextcloud-ddns.service -n 50 --no-pager
sudo journalctl -u nextcloud-cert-renew.service -n 50 --no-pager
sudo journalctl -u nextcloud-https.service -n 50 --no-pager
sudo journalctl -u nextcloud-data.service -n 50 --no-pager
trojan-vpn logs

sudo docker compose -f /srv/nextcloud-https/compose.yaml logs --tail 50 caddy
sudo docker compose --env-file /srv/nextcloud/.env \
  -f /srv/nextcloud/compose.yaml logs --tail 50 app go-vod
```

日志可能含用户文件名、路径、IP、订单 URL 等，分享前应裁剪和脱敏。不要开启 ACME debug 后把整份日志公开。

### 12.3 DDNS 手动检查、执行、暂停

```bash
# 只读检查，不写 DNS
sudo python3 /srv/nextcloud-https/ddns.py --check-only

# 本地模拟测试，不开启 VPN、不写 DNS
sudo python3 /srv/nextcloud-https/check-ddns.py

# 立即执行一次，有变化才更新
sudo systemctl start nextcloud-ddns.service

# 暂停自动调度；已在执行的 service 不会仅因此被中断
sudo systemctl disable --now nextcloud-ddns.timer

# 恢复调度
sudo systemctl enable --now nextcloud-ddns.timer
```

修改 service/timer 后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart nextcloud-ddns.timer
```

修改 Python 文件后，下次启动会加载新代码。编辑前做私有备份，同时检查配套 policy，避免生产脚本与 VPN 配置不匹配。

### 12.4 证书检查与代理重载

```bash
# 正常续期检查；未到期会跳过，不强制签发
sudo systemctl start nextcloud-cert-renew.service
sudo journalctl -u nextcloud-cert-renew.service -n 30 --no-pager

# 校验当前 Caddy 配置
sudo docker compose -f /srv/nextcloud-https/compose.yaml exec -T caddy \
  /usr/local/bin/caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile

# 检查证书及公私钥匹配后重载，不需要重启整套 Nextcloud
sudo systemctl reload nextcloud-https.service
```

禁止为了“试一下”反复 `--force` 签发；可能触发共享域名额度或失败验证限制。

### 12.5 必须重建单个 app 时

仅在已经批准变更且完成备份后：

```bash
sudo docker compose --env-file /srv/nextcloud/.env \
  -f /srv/nextcloud/compose.yaml config --quiet

sudo docker compose --env-file /srv/nextcloud/.env \
  -f /srv/nextcloud/compose.yaml up -d --no-deps \
  --wait --wait-timeout 180 app
```

此操作会短暂影响请求。升级应用镜像还需协调 cron 使用相同版本，不能机械套用“只重建 app”。升级 Nextcloud、MariaDB 或 Redis 前另做升级计划，不要直接 pull latest。

## 13. 密钥管理、备份与恢复范围

### 13.1 不得公开的文件

```text
/srv/nextcloud/.env
/srv/nextcloud/html/config/config.php
/srv/nextcloud/html/config/zz-migration.config.php
/srv/nextcloud-migration/secrets/
/srv/nextcloud-https/secrets/
/srv/nextcloud-https/acme-state/
/srv/nextcloud-https/acme-state-staging/
/srv/nextcloud-https/certs/key.pem
/srv/nextcloud-https/backups/
/etc/trojan-vpn/config.json
```

尤其不要把“整个 `/srv/nextcloud-https` 打包上传 GitHub”当成部署文档交付。它包含 Token、ACME 账户密钥、TLS 私钥以及带口令的备份。

`secrets/dynv6-token` 为 root:root 0600，父目录受限；token 名称为 `nextcloud-gavin-nas-https`，授权限定到当前 zone。

### 13.2 Token 轮换原则

1. 在 dynv6 创建仅授权本 zone 的新 token，不扩大为所有 zones。
2. 暂停 DDNS 和证书续期调度，并确认相关 service 已完成，避免轮换时并发读取。
3. 在 root 私有位置备份旧凭据；不要打印旧值。
4. 使用隐藏输入方式验证新 token，再原子替换凭据文件并保持 0600/root 所有权。
5. 现有 `store-dynv6-token.py` **拒绝覆盖已有文件**；它是初次配置 helper，不能无准备地当轮换器运行。
6. 确认 DDNS 只读检查成功、正式 ACME 使用新 token；wrapper 会从同一文件导出 DYNV6_TOKEN。ACME 内部还可能保存历史账户配置，不要公开它。
7. 恢复两个 timer，再撤销旧 token。

轮换可能影响 DNS 更新和证书续期，建议作为独立维护任务，不在无授权的故障排查中顺手执行。

### 13.3 现有备份与限制

历史迁移快照和记录主要在：

```text
旧机 /var/backups/nextcloud-migration/
新机 /srv/nextcloud-migration/source-snapshot/
新机 /srv/nextcloud-migration/final-20261006/
新机 /srv/nextcloud-migration/migration-state.json
```

HTTPS/路由变更前的私有备份示例：

```text
/srv/nextcloud-https/backups/nextcloud-compose.before.yaml
/srv/nextcloud-https/backups/config.before.php
/srv/nextcloud-https/backups/config.before-canonical-https.php
/srv/nextcloud-https/backups/nextcloud-compose.before-https-only.yaml
/srv/nextcloud-https/backups/ddns.before-vpn-direct.py
/srv/nextcloud-https/backups/trojan-config.before-ddns-direct.json
/srv/nextcloud-https/backups/trojan-config.before-0c7a07e02aea.json
```

这些是历史回滚材料，**不是已经验证的持续备份制度**。一些旧 Compose/配置仍包含 HTTP 入口或旧数据库路径；恢复时必须理解差异，不能整体覆盖现状。

用户数据、数据库、应用和身份配置必须形成一致的恢复点。不要单独恢复一个很旧的数据库到当前 USB 数据上，也不要在数据库在线时直接复制 MariaDB 数据目录冒充一致备份。

### 13.4 新机器恢复的必要前提

本文不足以独立重建生产系统，还需要私有材料：

- 有效的应用配置和数据库一致性备份，以及匹配的用户数据。
- 原始 Nextcloud 身份信息及 secrets；不能用“新装同版本”替代原实例。
- SMB、数据库、Trojan、dynv6/ACME 所需凭据。
- 自定义 Nextcloud/go-vod 镜像或完整可重建的构建上下文。
- 正式 TLS 证书/私钥，或重新执行合法 DNS-01 签发的能力。
- 先恢复受保护的数据挂载，再恢复主 Compose，最后恢复 HTTPS 和 timers。

不要直接运行 `provision.sh` 假定它会完成数据库恢复、迁移或所有后续加固。

## 14. 常见故障定位

| 现象 | 优先检查 | 不要做 |
| --- | --- | --- |
| `http://...:8081` 打不开 | 这是 HTTPS-only 后的预期行为 | 不要恢复 HTTP 映射 |
| 不带 `:8443` 的域名打不开 | 公网 443 历史上不可用，使用完整 HTTPS URL | 不要认为证书出错 |
| 域名无法解析，但 `--resolve` HTTPS 正常 | dynv6 A、家庭解析器、DDNS journal、权威 DNS | 不要立刻重签证书 |
| 域名指向旧家庭 IP | `.182` 是否在线、timer 是否启用、双源是否一致、API 是否成功 | 不要把 VPN IP 手工写入 A |
| DDNS service 是 inactive | 看上次 Result/退出码及 timer 状态 | 不要为 oneshot 强加常驻循环 |
| DDNS 报策略不匹配 | sing-box 实际配置与 `vpn_policy.py` 是否一致、服务是否已重载 | 不要删掉保护检查硬写 DNS |
| VPN 开启后 IP 查询异常 | 白名单精确域名、TLS sniff、ddns-direct、直连 DoH、网卡名 | 不要只靠 curl 绑定接口 |
| HTTPS 502 | app 是否就绪、Docker network、上游名字、数据挂载、DB/Redis | 不要只重载证书 |
| 数据目录缺失 / 容器拒绝启动 | 旧 `.114`、SMB、USB UUID、挂载和 `.ncdata` | 不要新建空数据目录或删除保护入口 |
| CLI 链接仍指向旧 HTTP | 用 occ 读取有效值，检查后加载的 config 覆盖文件 | 不要只看主 config.php |
| 视频有声音无画面 | 是否在独立 Memories；go-vod、FFmpeg、HLS、CPU 转码 | 不要批量重编码/覆盖原文件 |
| 证书续期失败 | 正式 CA endpoint、DNS API、TXT 传播、限额、reload hook | 不要用 staging 证书或 curl -k 掩盖 |

### 14.1 DNS 传播的历史问题

曾出现 dynv6 三个权威节点中一个缺失 A 的情况。2026-10-07 18:19 以后从 Azure 直接 TCP 查询三个节点均返回正确 A：

```text
95.216.144.82
159.69.43.243
5.45.100.251
```

这些是当时诊断用的节点 IP，不应写死为系统长期 DNS。此前家庭路由器 IPv4 DNS `192.168.1.1` 仍偶发 `NOERROR` 但无 A；紧接着系统正常解析和 HTTPS 又成功，尚未彻底定因。

域名已恢复并不等于免费 DNS 有 SLA。今后有故障要重新区分权威记录、缓存解析器、DDNS 进程、NAT 和应用层。

### 14.2 已知待办，不要说成已经完成

- 确认并记录 `.182` 的 DHCP 保留或静态地址策略。
- 物理搬迁 USB 到新机尚未完成；需要单独停写、切换挂载和一致性计划。
- 尚未建立并验证新的持续数据库/应用备份及异地恢复演练。
- 用户要求不备份 USB 的决定仍有效，但存在数据丢失风险。
- Memories 全量索引跳过的 2 个失败文件尚未逐项处理。
- 较早 Nextcloud setupchecks 曾有数据库检查、mimetype migration、日志/邮件等警告；本轮未全面重测，不能声称已全部解决。
- 对 HTTP 关闭后的媒体读取、HTTPS 和认证会话做过检查；没有为测试新建/上传用户文件，也没有执行完整写入负载测试。
- 没有为了验证这次配置而重启整台主机。已检查 enabled 单元和依赖，但不等价于完成断电恢复测试。
- 本轮实际证明了 VPN 开启时 DDNS 正常；没有长时间、跨多网络验证“VPN 长期开启时所有外部 Nextcloud 会话”这一更广泛场景。
- 失败会进入 journal/systemd 状态；未配置邮件、短信或即时消息告警。

## 15. 迁移历史：仅用于理解，不可直接执行

旧主机原先运行 ARM64 Nextcloud 18.0.4；后经阶段交接和逐个主要版本升级到当前 35.0.1。用户选择 Docker Compose，而非 Kubernetes。

旧交接文件：

```text
/home/xlucky/nextcloud-migration-handoff.md
/srv/nextcloud-migration/HANDOFF.md
```

其中部分内容停留在 Nextcloud 18.0.14 阶段，不是当前运行状态。以本文件的复核时间和主机实际状态为准。

迁移时重点保留账号、分享、评论、通讯录、活动、版本和回收站。历史基线曾包括 3 个用户、8 个分享、49 条评论、3 个联系人、978 个版本文件和 2533 个回收站文件。

这些是**历史迁移检查点**，不是现在必须完全相等的计数；用户后续使用会使数据变化。不能仅因为今天不同就判定迁移失败，也不能未经核对宣称新一轮完整一致性审计已完成。

## 16. 文件指纹与来源

以下为 2026-10-07 在服务器上读取的 SHA256，用于后续识别配置漂移，不代表自动信任以后任何同名文件：

| `/srv/nextcloud-https/` 内文件 | SHA256 |
| --- | --- |
| `ddns.py` | `f4ac9002807c107003778c03fcf06e5c8316581504836c060da2694753fc31f3` |
| `vpn_policy.py` | `e9547efeda177f308217e12a92eb7760fbc862129b3197870da60ae717669b8d` |
| `certificate.sh` | `4bc40c3aae0207c735abc5e783f2bca6f60fd8d2653bd39e350b5f038a8ed5e9` |
| `reload.sh` | `c9b02217f37c85ab148cfe649a6a059b3c681b98eced8975aefca7e2ce856f7e` |
| `Caddyfile` | `356d59eb18cb9784844adc1101fab5f93184fca3183c33fc206d9e08656b2e0a` |
| `compose.yaml` | `db160018f981ed26bec760baa87df90bcec20cbea5926f49bae18b0e115286c8` |

官方参考：

- Nextcloud 反向代理：<https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/reverse_proxy_configuration.html>
- dynv6 API：<https://dynv6.github.io/api-spec/>
- acme.sh：<https://github.com/acmesh-official/acme.sh>
- Caddy：<https://github.com/caddyserver/caddy>
- sing-box route action：<https://sing-box.sagernet.org/configuration/route/rule_action/>
- Let’s Encrypt 限额：<https://letsencrypt.org/docs/rate-limits/>

## 17. 建议给新 LLM session 的开场任务

> 请先阅读 `nas/HAPPY-FAMILY-DEPLOYMENT.md`，把它作为 2026-10-07 的部署快照而非实时事实。
>
> 先做只读检查并报告与文档的差异：Nextcloud 版本、maintenance/needsDbUpgrade、SMB 实际来源、Docker 端口发布、HTTPS 证书、DDNS/续期 timers 和 Trojan 当前状态。
>
> 不要输出凭据，不要恢复 HTTP，不要重跑历史迁移，不要启动旧主机的旧 Nextcloud，不要删除卷或修改原始媒体。USB 仍依赖 `.114`；默认不要开启 VPN。若任务确需改配置或验证 VPN 开启，先说明影响并获得授权。
>
> 最后只执行本次用户明确要求的变更，验证结果，更新本文件的状态日期、相关章节和未完成事项；不要把历史检查结果写成新的实测结果。

---

维护本文时，优先更新“当前配置/待办/复核日期”，保留有用的故障背景。不要把调试过程中的临时地