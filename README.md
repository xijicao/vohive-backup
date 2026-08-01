# VoHive 离线备份与恢复

这是一个私人备份仓库，保存 VoHive 的 Linux、Docker 和 iStoreOS 版本，方便原镜像或下载源失效后恢复。

## 已备份内容

| 目录 | 内容 | 版本 |
|---|---|---|
| `linux/` | 原版 Linux 可执行文件，支持 amd64 和 arm64 | 1.5.5 |
| `docker/` | 使用上述原版二进制自行构建 Docker 镜像 | 1.5.5 |
| `istore/` | iStoreOS/OpenWrt 核心和 LuCI 插件 | 核心 1.5.4，LuCI 0.1.18 |

> HK1Box X3 是 **arm64**，安装 iStoreOS 包时不要误装 `amd64`。

## 一、先校验文件

Linux：

```bash
sha256sum -c SHA256SUMS.txt
```

Windows PowerShell 可逐个检查：

```powershell
Get-FileHash -Algorithm SHA256 .\linux\vohive_v1.5.5_linux_arm64
Get-FileHash -Algorithm SHA256 .\linux\vohive_v1.5.5_linux_amd64
```

两份原版 1.5.5 的已知哈希：

```text
arm64  615bb2b20fadf6f7d269ec455d93a09a5f97d358ff095edda7211fae6b56b259
amd64  3dd53751eaba8aa3a30a312f65a5ad950b6ba2e677254d2f5608c4d9228a6df7
```

## 二、Docker 使用方法（推荐）

要求：Linux 主机已安装 Docker Compose，USB 模组已正常枚举为 `/dev/ttyUSB*` 或 QMI 设备。

```bash
git clone git@github.com:xijicao/vohive-backup.git
cd vohive-backup/docker

# 首次运行：复制示例配置，并立刻修改密码
cp config/config.yaml.example config/config.yaml
nano config/config.yaml

# 从仓库内的原版二进制构建并启动
docker compose up -d --build

# 查看状态和日志
docker compose ps
docker compose logs -f vohive
```

浏览器访问 `http://设备IP:7575`。不要把 7575 端口直接映射到公网。

如果 GitHub Actions 已成功构建 GHCR 镜像，也可把 `compose.yaml` 中的 `build:` 删除，然后把 `image:` 改为：

```yaml
image: ghcr.io/xijicao/vohive-backup:v1.5.5
```

因为仓库和镜像是私有的，首次拉取前需要登录：

```bash
# 密码位置输入只带 read:packages 权限的 GitHub PAT，不要把 PAT 写进文件
docker login ghcr.io -u xijicao
docker pull ghcr.io/xijicao/vohive-backup:v1.5.5
```

### Docker 常见问题

- 模组被占用：先检查 `systemctl status ModemManager`。如它占用了模组，应让它忽略该设备，或在确认该主机不需要它管理其他蜂窝设备后再停用。
- 看不到 USB：确认 `lsusb`、`ls -l /dev/ttyUSB* /dev/cdc-wdm*`，然后重插模块。
- 容器反复重启：确认 `docker/config/config.yaml` 已存在且 YAML 缩进正确。
- 修改配置：先 `docker compose stop vohive`，改完后执行 `docker compose up -d`，避免程序同时写配置。

## 三、直接在 Linux 运行

arm64（树莓派、ARM 盒子等）：

```bash
sudo mkdir -p /opt/vohive/{config,data,logs}
sudo install -m 0755 linux/vohive_v1.5.5_linux_arm64 /opt/vohive/vohive
sudo cp docker/config/config.yaml.example /opt/vohive/config/config.yaml
sudo nano /opt/vohive/config/config.yaml
cd /opt/vohive
sudo ./vohive -c ./config/config.yaml
```

amd64（常见 Intel/AMD 小主机）：把上面文件名换成 `vohive_v1.5.5_linux_amd64`。

确认前台运行正常后，可安装 systemd 服务：

```bash
sudo cp linux/vohive.service.example /etc/systemd/system/vohive.service
sudo systemctl daemon-reload
sudo systemctl enable --now vohive
sudo systemctl status vohive
```

## 四、iStoreOS / OpenWrt 恢复

当前备份的是你已经保存的版本：核心 1.5.4、LuCI 0.1.18。它不是 Linux 1.5.5 的同版本 IPK，请不要混淆。

HK1Box X3（arm64）通过 SCP 上传两个文件到 `/tmp` 后：

```bash
# 先备份当前配置；命令不会删除原配置
cp -a /etc/config/vohive /root/vohive-config-backup 2>/dev/null || true

opkg install /tmp/vohive-core-arm64_1.5.4-r1_all.ipk
opkg install /tmp/luci-app-vohive_0.1.18-r1_all.ipk
```

Intel/AMD x86_64 软路由才使用：

```bash
opkg install /tmp/vohive-core-amd64_1.5.4-r1_all.ipk
opkg install /tmp/luci-app-vohive_0.1.18-r1_all.ipk
```

不要在工作正常的设备上用 `--force-reinstall` 盲目覆盖。需要恢复时先备份 `/etc/config/vohive` 以及 VoHive 数据目录。

## 五、安全和隐私

- 示例密码只是占位符，启动前必须修改。
- VoHive 能读取 IMSI、ICCID、短信和设备信息；不要提交真实 `config.yaml`、日志或数据目录。
- 本仓库只提交 `config.yaml.example`，`.gitignore` 会阻止本地真实配置、日志和数据被误传。
- Docker 需要访问 USB/QMI 设备，因此示例使用 `privileged`、host 网络和 `/dev` 透传；仅在可信主机和局域网使用。

## 六、来源说明

Linux 1.5.5 二进制来自社区保存的 [pexcn/docker-images VoHive Release](https://github.com/pexcn/docker-images/releases/tag/vohive)，下载后已按发布页 SHA256 校验。本仓库用于个人备份和灾难恢复，不声明对 VoHive 程序本身的著作权或授权。

