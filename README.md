# 把 ARMv7 电视盒子变成短信通知网关：Asterisk、Quectel 与 Bark 部署教程

把闲置电视盒子刷成 Linux，接上移远 Quectel 模块，就可以用 Asterisk 接收短信，再通过 Bark 推送到手机。

示例采用华为悦盒 EC6108V9C / Hi3798MV100 类 ARMv7 设备、Quectel EC20F/EC25 系列模块和 `2c7c:0125` USB ID。实际板型、无线芯片、模块固件和端口编号可能不同，部署前应以设备探测结果为准。

## 1. 准备硬件，理解各部分的职责

| 项目     | 本文使用的环境或约定                              |
| ------ | --------------------------------------- |
| 电视盒子   | 华为悦盒 EC6108V9C，实际 SoC、板型以设备为准           |
| 系统     | HiNAS 32-bit，Ubuntu 20.04.6 LTS         |
| 内核与架构  | `4.4.35_ecoo_81092768`，`armv7l`         |
| 蜂窝模块   | Quectel EC20 / EC25 家族中的兼容型号，需核对实际固件与接口 |
| USB 标识 | 本例 `2c7c:0125`，以 `lsusb` 的实际结果为准        |
| 供电     | 建议使用带独立供电的 USB Hub，保证模块供电稳定             |
| 电脑     | Windows、macOS 或 Linux；首次配置建议保留网线连接      |
| 通知客户端  | 已安装 Bark 的 iPhone，以及自己的 Bark Device Key |

刷机前必须核对具体板型，不能仅凭盒子的外壳名称选择固件。Hi3797 与 Hi3798 的内核、Wi-Fi 驱动和刷机方式可能不同。

### 已验证的运行形态

完成部署后，可以用下面的检查结果判断系统是否处于同一类状态：

| 项目       | 预期结果                                                         |
| -------- | ------------------------------------------------------------ |
| 主机       | Ubuntu 20.04.6、ARMv7、旧版 4.4 内核                               |
| 无线接口     | `wlan0` 由 NetworkManager 管理；有线接口可作为救援链路                      |
| 蜂窝模块     | `lsusb` 显示 `2c7c:0125`，`option` 驱动绑定到 `ttyUSB0` 至 `ttyUSB4`  |
| Docker   | `asterisk-quectel:armv7` 镜像、`ast-test` 容器、host 网络、只映射五个串口    |
| Asterisk | 20.6.0，`chan_quectel.so` 状态为 `Running`                       |
| 配网       | `box-provision.service` 监听 `127.0.0.1:8765`，Nginx 只对配置热点网段代理 |


整个系统可以按两条流程理解：

```text
短信：SIM 卡 → Quectel 模块 → 宿主机 option 驱动 → ttyUSB
     → Docker 设备映射 → chan_quectel → Asterisk → Bark → 手机

管理：通电 → 尝试已保存的 Wi-Fi → 成功：Bark 通知 IP
                            └→ 失败：开启 Box-Setup 热点 → 网页配网
```

```mermaid
flowchart LR
    M[Quectel EC20/EC25] --> U[option 驱动]
    U --> T[/dev/ttyUSB0..4]
    T --> D[Docker ast-test]
    D --> Q[chan_quectel]
    Q --> A[Asterisk]
    A --> B[Bark 短信推送]
    W[NetworkManager wlan0] --> P[box-provision]
    P --> N[Nginx :80]
    N --> H[Box-Setup 192.168.50.1]
    P --> B
    S[systemd] --> U
    S --> D
    S --> A
```

USB 驱动由宿主机内核管理。容器只能使用宿主机已经识别出的设备，因此不能把 `echo ... > new_id` 放进容器内部，再指望它修复容器创建时缺少串口的问题。

本文先完成短信网关，再安装可选的网页配网服务。所有 Linux 命令默认在 **Bash** 中运行；安装服务、写入 `/etc` 与绑定 USB 的步骤均在盒子的 root shell 中执行。Windows 下构建镜像可使用 WSL 的 Bash，Docker Desktop 需切换到 Linux 容器。

## 2. 刷机并建立第一次 SSH 连接

固件和具体刷机方法参考平台文档：

- [固件下载](https://www.ecoo.top/download)
- [机顶盒刷机教程](https://www.ecoo.top/docs/category/%E6%9C%BA%E9%A1%B6%E7%9B%92%E5%88%B7%E6%9C%BA%E6%95%99%E7%A8%8B)

刷机完成后，可以把盒子通过网线连接到电脑，并将电脑的 Wi-Fi 连接共享给以太网。Windows 网络共享常使用 `192.168.137.0/24`，但盒子的地址由 DHCP 分配，不能固定照抄别人的地址。

在 Windows PowerShell 查看相邻设备：

```powershell
Get-NetNeighbor -AddressFamily IPv4
arp -a
```

找到盒子的 IP 后连接。以下地址只是示例，请替换：

```bash
ssh root@192.168.137.100
```

登录后记录系统信息，并设置自己的登录密码：

```bash
uname -a
uname -m
cat /etc/os-release
ip -br address
passwd
```

安装基础工具。设备已有 Docker 时保留现有安装，不需要重复安装另一套 Docker：

```bash
apt update
apt install -y kmod usbutils curl ca-certificates python3 jq \
  network-manager openssh-server avahi-daemon nginx git
systemctl enable --now ssh NetworkManager avahi-daemon
docker version
docker info
```

如果 `docker` 尚未安装，优先采用该固件说明中适配 ARMv7 和旧内核的安装方式。发行版仓库提供兼容包时，可使用 `apt install docker.io`，然后执行 `systemctl enable --now docker`。仅有 Docker 客户端并不足够，`docker info` 必须能连接服务端。

构建和运行镜像前确认架构没有被误识别为 `amd64`：

```bash
docker version --format 'server={{.Server.Version}} client={{.Client.Version}}'
docker info --format 'arch={{.Architecture}} os={{.OperatingSystem}} kernel={{.KernelVersion}}'
```

本文验证设备使用 Docker 26.1.3、`armv7l` 和 4.4.35 内核。旧内核能运行较新的 Docker，并不代表每个新镜像都兼容；遇到容器启动异常时应先检查 libc、设备权限和内核特性。

第一次登录使用临时密码即可。完成公钥登录验证后，应关闭 root 密码登录或至少限制 SSH 来源网段：

```bash
install -d -m 0700 /root/.ssh
chmod 0600 /root/.ssh/authorized_keys
sshd -t
```

修改 `sshd_config` 前保留一个已登录的救援会话，并在新会话中验证密钥；不要在唯一 SSH 连接里直接重启 SSH。

### 2.1 确认 Wi-Fi 驱动和连接配置

```bash
nmcli device status
ip -br link
```

如果固件缺少 Wi-Fi 驱动，可以检查针对 Hi3798MV100 的驱动脚本。先阅读安装脚本，确认板型与内核适配，再执行：

```bash
git clone https://gitee.com/xjxjin/scripts.git
cd scripts
less install_hi3798mv100_wifi.sh
sudo ./install_hi3798mv100_wifi.sh
sudo depmod -a
```

不要把某个设备的 `/lib/modules/4.4.35_...` 路径硬编码到所有盒子。当前模块目录应与 `uname -r` 一致；仅创建目录并不能补齐缺失的驱动。

首次可以通过网线 SSH 配置无线网络：

```bash
nmcli device wifi rescan ifname wlan0
nmcli device wifi list ifname wlan0
nmcli --ask device wifi connect 'MyHomeWiFi' ifname wlan0 name home-wifi
nmcli connection modify home-wifi connection.autoconnect yes
nmcli connection modify home-wifi ipv4.route-metric 50
```

`MyHomeWiFi` 是示例 SSID，`home-wifi` 是 NetworkManager 的连接配置名，两者不必相同。`connection modify` 接收的是配置名或 UUID。路由 metric 越小优先级越高；是否让 Wi-Fi 优先于网线，应按实际网络选择。

改完连接属性通常在下次激活时生效。远程配置期间保留网线，再执行 `nmcli connection up home-wifi`；直接重启整个 NetworkManager 可能使当前 SSH 中断。

## 3. 先让宿主机出现 ttyUSB

### 3.1 区分 USB 枚举与串口绑定

先检查模块是否已被 USB 总线识别：

```bash
lsusb
lsusb -t
dmesg | tail -80
```

本例设备的 VID/PID 为 `2c7c:0125`，实测设备在 Asterisk 中显示为 EC20F。仅凭 USB ID 仍不足以确认硬件变体，必要时可通过模块标签或 AT 命令 `ATI` 核对。

如果 `lsusb` 中已经存在该设备，但没有 `/dev/ttyUSB*`，在**宿主机**执行：

```bash
modprobe usbserial
modprobe option
echo 2c7c 0125 > /sys/bus/usb-serial/drivers/option1/new_id
sleep 2
ls -l /dev/ttyUSB*
dmesg | tail -50
```

本次实机最终出现了 `/dev/ttyUSB0` 到 `/dev/ttyUSB4`。其他固件的 USB 接口组合可能不同，不能把“五个端口”当作所有 EC20 / EC25 的固定规格。

`new_id` 用于向运行中的驱动注册设备 ID，这个动态注册不能替代持久化启动配置。也不要用 `cat .../new_id` 判断是否绑定成功；应检查字符设备、驱动链接和内核日志。

如果使用普通用户，重定向也需要 root 权限：

```bash
printf '2c7c 0125\n' | sudo tee /sys/bus/usb-serial/drivers/option1/new_id >/dev/null
```

**如果 `lsusb` 根本看不到模块，先检查供电、数据线和 USB Hub。** 某些设备的 USB 模块可能在开机很久以后才完成枚举；写入 `new_id` 无法让一个尚未枚举的 USB 设备凭空出现，应先从硬件和内核 USB 日志排查。

### 3.2 写一个每次启动都执行的准备脚本

创建 `/usr/local/sbin/quectel-usb-prepare.sh`：

```bash
cat > /usr/local/sbin/quectel-usb-prepare.sh <<'EOF'
#!/bin/sh
set -eu
PATH=/usr/sbin:/usr/bin:/sbin:/bin

# Vendor kernels may have built-in drivers without separate module files.
modprobe usbserial 2>/dev/null || true
modprobe option 2>/dev/null || true
new_id=/sys/bus/usb-serial/drivers/option1/new_id
if [ ! -w "$new_id" ]; then
    echo "Quectel: option driver new_id is unavailable" >&2
    exit 1
fi

# Always register the VID/PID before starting the container.
# Some kernels reject duplicate registrations; readiness is checked below.
if ! { echo 2c7c 0125 > "$new_id"; } 2>/dev/null; then
    echo "Quectel: new_id write did not succeed; checking existing binding" >&2
fi

attempt=0
while [ "$attempt" -lt 30 ]; do
    ready=1
    for port in 0 1 2 3 4; do
        dev="/dev/ttyUSB$port"
        if [ ! -c "$dev" ]; then
            ready=0
            break
        fi
        props=$(udevadm info --query=property --name="$dev" 2>/dev/null || true)
        if ! printf '%s\n' "$props" | grep -qx 'ID_VENDOR_ID=2c7c' ||
           ! printf '%s\n' "$props" | grep -qx 'ID_MODEL_ID=0125'; then
            ready=0
            break
        fi
    done
    if [ "$ready" -eq 1 ]; then
        echo "Quectel: ttyUSB0..4 are ready"
        exit 0
    fi
    attempt=$((attempt + 1))
    sleep 1
done
echo "Quectel: required serial devices did not become ready" >&2
exit 1
EOF
chmod 0755 /usr/local/sbin/quectel-usb-prepare.sh
/usr/local/sbin/quectel-usb-prepare.sh
```

脚本每次都会尝试执行核心的 `echo`，然后检查五个串口是否为字符设备、是否属于目标 VID/PID。某些厂商内核把驱动直接编入内核，因此以 `new_id` 接口实际存在为准，而不只看 `modprobe` 的返回值。等待约 30 秒仍不满足条件时返回失败，由后面的 systemd 服务重试。

这里假定只连接一只蜂窝模块，而且端口固定为 `ttyUSB0–4`。如果有多只 USB 串口设备，应根据 `/dev/serial/by-id/` 或 USB 接口号建立稳定别名，并同步修改准备脚本、容器映射和 `quectel.conf`，避免端口编号变化后连接错设备。

## 4. 构建 ARMv7 的 Asterisk + chan_quectel 镜像

这一节提供一种可复建的参考路径：使用 Debian Bullseye 的 Asterisk 与匹配的开发头文件，在容器中编译 `chan_quectel`。它不要求在电视盒子上编译整套 Asterisk。目标设备上实际运行的是 Asterisk 20.6.0，因此构建时应让 `--with-astversion` 与镜像里的版本完全一致。

本文使用公开的 [IchthysMaranatha/asterisk-chan-quectel](https://github.com/IchthysMaranatha/asterisk-chan-quectel) 作为构建示例，并固定一个提交。生产环境应把经过测试的提交、基础镜像 digest 和 Asterisk 版本一起记录。

### 4.1 创建 Dockerfile

在用于构建镜像的电脑或 ARMv7 设备上执行：

```bash
mkdir -p ast-build
cd ast-build
cat > Dockerfile <<'EOF'
FROM debian:bullseye
ARG DEBIAN_FRONTEND=noninteractive
ARG QUECTEL_REF=3d45c7f072131296a7e3c1a4faf5bb18751dbd87

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      asterisk asterisk-dev python3 ca-certificates curl \
      libasound2 libsqlite3-0 \
      build-essential autoconf automake libtool pkg-config git \
      libsqlite3-dev libasound2-dev \
 && git clone https://github.com/IchthysMaranatha/asterisk-chan-quectel.git /usr/src/chan-quectel \
 && cd /usr/src/chan-quectel \
 && git checkout --detach "$QUECTEL_REF" \
 && ./bootstrap \
 && AST_VERSION="$(asterisk -V | sed -n 's/^Asterisk \([0-9][0-9.]*\).*/\1/p')" \
 && test -n "$AST_VERSION" \
 && ./configure --with-astversion="$AST_VERSION" \
      --with-asterisk=/usr/include DESTDIR=/usr/lib/asterisk/modules \
 && make -j2 \
 && make install \
 && mkdir -p /usr/local/share/quectel /run/asterisk /etc/box-provision \
 && cp etc/quectel.conf /usr/local/share/quectel/quectel.conf.sample \
 && printf '%s\n' "$QUECTEL_REF" > /usr/local/share/quectel/source-revision \
 && dpkg-query -W asterisk > /usr/local/share/quectel/asterisk-package-version \
 && apt-get purge -y --auto-remove \
      asterisk-dev build-essential autoconf automake libtool pkg-config git \
      libsqlite3-dev libasound2-dev \
 && rm -rf /usr/src/chan-quectel /var/lib/apt/lists/*

# systemd on the host starts and supervises Asterisk through docker exec.
CMD ["tail", "-f", "/dev/null"]
EOF
```

`--with-astversion` 必须对应容器内实际安装的 Asterisk 版本，不能把 README 的示例版本原样写死。`libasound2-dev` 用于构建该分支包含的 ALSA 支持，运行时保留 `libasound2`。

此 Dockerfile 固定了驱动提交，但没有固定基础镜像 digest 和发行版软件包版本。它会使用 Debian Bullseye 软件源提供的 Asterisk 版本；构建后必须用 `docker run --rm asterisk-quectel:armv7 asterisk -V` 核对版本，并让 `--with-astversion` 与该输出保持一致。目标环境已经验收的是 Asterisk 20.6.0；如果构建机得到其他版本，不要把它当作 20.6.0 的可替代镜像，应改用固定的 Asterisk 20.6.0 包或源码重新构建。需要长期重复构建时，还应保存基础镜像 digest 和发行版软件包版本；升级 Asterisk 后也要重新构建匹配的驱动。

### 4.2 选择正确的构建平台

在原生 ARMv7 Linux 上可以直接构建：

```bash
docker build -t asterisk-quectel:armv7 .
```

在 Windows / macOS 的 Docker Desktop 或已配置 ARM 模拟器的 Linux 构建机上：

```bash
docker buildx inspect --bootstrap
docker buildx build --platform linux/arm/v7 --load \
  -t asterisk-quectel:armv7 .
docker image inspect asterisk-quectel:armv7 \
  --format '{{.Os}}/{{.Architecture}}/{{.Variant}}'
```

构建器需要支持 `linux/arm/v7`。仅使用 `debian:bullseye` 不会自动让 x86 电脑生成 ARM 镜像；`--platform` 也不会自动解决所有构建机的模拟器配置问题。旧版 Docker 不支持 Buildx 时，可使用原生 ARMv7 机器构建。

将镜像传给盒子。先设置实际地址：

```bash
BOX_ADDR=192.168.137.100
docker save asterisk-quectel:armv7 | gzip > asterisk-quectel-armv7.tar.gz
scp asterisk-quectel-armv7.tar.gz "root@$BOX_ADDR:/root/"
```

在盒子上导入：

```bash
gzip -dc /root/asterisk-quectel-armv7.tar.gz | docker load
docker run --rm asterisk-quectel:armv7 asterisk -V
```

此处验证版本的临时容器不使用 USB，所以无需绑定串口。后续启动使用蜂窝模块的 `ast-test` 容器才必须先准备 USB。

## 5. 准备持久化配置和 Bark 通知

### 5.1 建立数据目录

下列步骤面向新部署。已有 `ast-test` 的设备应先备份现有配置和容器信息，不要直接用新配置覆盖。

```bash
install -d -m 0755 /opt/asterisk/etc /opt/asterisk/bin
install -d -m 0755 /opt/asterisk/lib /opt/asterisk/log /opt/asterisk/spool
install -d -m 0700 /etc/box-provision

docker create --name ast-config-seed asterisk-quectel:armv7
docker cp ast-config-seed:/etc/asterisk/. /opt/asterisk/etc/
docker cp ast-config-seed:/var/lib/asterisk/. /opt/asterisk/lib/
docker cp ast-config-seed:/var/spool/asterisk/. /opt/asterisk/spool/
docker rm ast-config-seed
```

先从镜像取出默认配置，再绑定目录。直接把一个空目录挂到 `/etc/asterisk`，会遮住镜像里原有的全部配置。

### 5.2 保存 Bark Key

在盒子的 Bash 中输入自己的 Key，输入过程不回显：

```bash
read -r -s -p 'Bark Device Key: ' BARK_DEVICE_KEY
printf '\n'
test -n "$BARK_DEVICE_KEY" && \
  (umask 077; printf '%s\n' "$BARK_DEVICE_KEY" > /etc/box-provision/bark-key)
unset BARK_DEVICE_KEY
```

Key 单独保存在宿主机，并以只读目录挂载进入容器。不要将真实 Key 写进 Dockerfile、截图或准备公开的文章。

### 5.3 安装通用推送脚本

创建 `/opt/asterisk/bin/bark-notify`，供宿主机的联网通知和容器内的短信通知共用：

```bash
cat > /opt/asterisk/bin/bark-notify <<'EOF'
#!/usr/bin/python3
import base64
import json
import sys
import time
from pathlib import Path
from urllib.request import Request, urlopen

def main():
    if len(sys.argv) != 4:
        raise ValueError("usage: bark-notify TITLE BODY GROUP | --sms SENDER BASE64")
    if sys.argv[1] == "--sms":
        sender, encoded = sys.argv[2:4]
        body = base64.b64decode(encoded, validate=True).decode("utf-8", errors="replace")
        title, group = "短信：" + (sender or "未知号码"), "SMS"
    else:
        title, body, group = sys.argv[1:4]
    key = Path("/etc/box-provision/bark-key").read_text(encoding="utf-8").strip()
    if not key:
        raise ValueError("empty Bark key")
    payload = json.dumps({
        "device_key": key, "title": title, "body": body,
        "group": group, "ttl": 600,
    }, ensure_ascii=False).encode("utf-8")
    for attempt in range(3):
        try:
            request = Request("https://api.day.app/push", data=payload,
                              headers={"Content-Type": "application/json"}, method="POST")
            with urlopen(request, timeout=10) as response:
                result = json.load(response)
            if result.get("code") != 200:
                raise RuntimeError("Bark did not accept the notification")
            return
        except Exception:
            if attempt == 2:
                raise
            time.sleep(2 ** attempt)

if __name__ == "__main__":
    try:
        main()
    except Exception as exc:
        # Keep keys, message bodies and phone numbers out of diagnostic logs.
        print("Bark notification failed: " + type(exc).__name__, file=sys.stderr)
        sys.exit(1)
EOF
chmod 0755 /opt/asterisk/bin/bark-notify
ln -sfn /opt/asterisk/bin/bark-notify /usr/local/bin/bark-notify
/usr/local/bin/bark-notify '盒子通知测试' 'Bark 配置成功' box
```

使用 JSON 序列化可以正确处理短信中的双引号、换行和中文；直接把短信插入 shell 拼接的 JSON 容易生成无效请求。脚本同时检查 HTTP 请求和 Bark 返回的业务状态，避免将失败误报为成功。

`group` 用于通知分组，`ttl` 是推送有效期相关参数，不是本地重试时间，也不等于短信自动删除时间。这个简化脚本最多尝试三次，没有持久化投递队列；需要可靠补发时应再增加本地队列与投递记录。

不要把 Key 写在 `notify.sh`、Asterisk 配置或镜像层中。已经这样部署过的镜像即使删除脚本中的字符串，也不能撤销已经泄露的凭据；应先在 Bark 端轮换 Key，再重建镜像或用只读文件挂载新 Key。

## 6. 配置 chan_quectel 和短信拨号计划

### 6.1 选择数据端口与音频端口

EC20/EC25 的常见配置为：

```ini
data=/dev/ttyUSB2
audio=/dev/ttyUSB1
```

这是接口示例，不代表所有模块都按此编号。`data` 是 AT 命令端口，`audio` 是相应固件支持的串行音频接口；UAC 模式还需要音频设备和对应配置。本文以短信接收为目标，不保证语音通话、VoLTE 或特定音频模式已可用。

创建 `/opt/asterisk/etc/quectel.conf`：

```bash
cat > /opt/asterisk/etc/quectel.conf <<'EOF'
[general]
interval=15
smsdb=/var/lib/asterisk/smsdb
csmsttl=600

[defaults]
context=incoming-mobile
group=0
rxgain=0
txgain=0
disablesms=no
autodeletesms=no
resetquectel=yes
u2diag=-1
usecallingpres=yes
callingpres=allowed_passed_screen
language=en
initstate=start

[quectel0]
data=/dev/ttyUSB2
audio=/dev/ttyUSB1
context=from-quectel
EOF
```

这里先使用 `autodeletesms=no`，便于调试期间保留模块中的短信。长期运行需要管理短信存储容量；改为 `yes` 后，驱动删除短信与 Bark 是否成功送达不是一项原子操作，不能把它当作可靠消息队列。不要照抄模块返回的 IMEI/IMSI 到配置文件，驱动无法识别的 `auto` 值会产生警告，省略这两项更稳妥。

### 6.2 只将过滤后的参数传给外部脚本

创建 `/opt/asterisk/etc/extensions.conf`。以下内容会替换新部署的默认拨号计划：

```bash
cat > /opt/asterisk/etc/extensions.conf <<'EOF'
[general]
static=yes
writeprotect=yes

[from-quectel]
exten => sms,1,NoOp(Forward incoming SMS to Bark)
 same => n,Set(SAFE_SENDER=${FILTER(0-9+,${CALLERID(num)})})
 same => n,Set(SMS_B64=${BASE64_ENCODE(${SMS})})
 same => n,System(/usr/local/bin/bark-notify --sms "${SAFE_SENDER}" "${SMS_B64}")
 same => n,NoOp(Bark command result: ${SYSTEMSTATUS})
 same => n,Hangup()

; This example does not route voice calls or send outgoing SMS.
exten => s,1,Hangup()
exten => _X.,1,Hangup()
exten => ussd,1,Hangup()
EOF
```

当前驱动把短信正文放在 `${SMS}`，拨号计划先用 `BASE64_ENCODE()` 转成只含安全字符的参数，再交给 Python 脚本解码并生成 JSON。不要把 `${SMS}` 或 `${BASE64_DECODE(...)}` 解码后的短信正文直接插入 `System()`：短信属于外部输入，可能包含 shell 特殊字符。

本例过滤号码后，带中文或标点的发送者名称可能无法完整保留；如需保留完整字段，建议改为 AGI，通过协议传递数据，避免将未过滤内容拼进 shell。

### 6.3 控制加载的模块和配置权限

在新部署的 `/opt/asterisk/etc/modules.conf` 中使用：

```ini
[modules]
autoload=yes
load=chan_quectel.so
noload=chan_sip.so
noload=chan_pjsip.so
noload=res_pjsip.so
noload=chan_iax2.so
noload=res_http_websocket.so
noload=cdr_sqlite3_custom.so
noload=cel_sqlite3_custom.so
```

确保 `/opt/asterisk/etc/manager.conf` 的 `[general]` 中是 `enabled=no`，`/opt/asterisk/etc/http.conf` 的 `[general]` 中也是 `enabled=no`。短信网关不需要对外开放 AMI、ARI、SIP 或 IAX 服务。`autoload=yes` 下不同版本仍可能加载其他模块，所以启动后要实际检查监听端口。

```bash
chown -R root:root /opt/asterisk/etc /opt/asterisk/bin /etc/box-provision
find /opt/asterisk/etc -type d -exec chmod 0755 {} +
find /opt/asterisk/etc -type f -exec chmod 0644 {} +
chmod 0700 /etc/box-provision
chmod 0600 /etc/box-provision/bark-key
```

系统启动时可能出现 `cdr_sqlite3_custom`、`cel_sqlite3_custom` 拒绝加载，同时仍然出现 `Asterisk Ready`。这两条错误不能单独证明 Asterisk 启动失败；不使用这类通话记录后端时，可以明确禁用。

## 7. 创建容器，由 systemd 负责启动顺序

### 7.1 创建使用 USB 的业务容器

先运行准备脚本，成功后再创建容器：

```bash
/usr/local/sbin/quectel-usb-prepare.sh && \
docker create --name ast-test \
  --restart=no \
  --network host \
  --device=/dev/ttyUSB0:/dev/ttyUSB0:rwm \
  --device=/dev/ttyUSB1:/dev/ttyUSB1:rwm \
  --device=/dev/ttyUSB2:/dev/ttyUSB2:rwm \
  --device=/dev/ttyUSB3:/dev/ttyUSB3:rwm \
  --device=/dev/ttyUSB4:/dev/ttyUSB4:rwm \
  --mount type=bind,src=/opt/asterisk/etc,dst=/etc/asterisk,readonly \
  --mount type=bind,src=/opt/asterisk/lib,dst=/var/lib/asterisk \
  --mount type=bind,src=/opt/asterisk/log,dst=/var/log/asterisk \
  --mount type=bind,src=/opt/asterisk/spool,dst=/var/spool/asterisk \
  --mount type=bind,src=/opt/asterisk/bin/bark-notify,dst=/usr/local/bin/bark-notify,readonly \
  --mount type=bind,src=/etc/box-provision/bark-key,dst=/etc/box-provision/bark-key,readonly \
  --log-driver json-file --log-opt max-size=5m --log-opt max-file=3 \
  asterisk-quectel:armv7
```

这里不使用 `--privileged`，只映射需要的串口。`--network host` 沿用原设备的部署方式，容器共享宿主机网络，内部监听的服务也可能直接暴露在盒子地址上，因此前面的服务关闭和端口检查不能省略。

`--restart=no` 是启动顺序的一部分：由 systemd 在 USB 准备成功之后启动业务容器，避免 Docker 自己先执行自动重启。已有容器可以先通过 `docker inspect` 查看，再用 `docker update --restart=no ast-test` 统一交给 systemd 管理。

设备上的工作容器名为 `ast-test`，镜像是 `asterisk-quectel:armv7`，命令是 `tail -f /dev/null`，Asterisk 由宿主机的 systemd 通过 `docker exec` 以前台模式运行。容器使用 host 网络、没有启用 `--privileged`，并映射 `/dev/ttyUSB0` 至 `/dev/ttyUSB4`。这种“容器提供用户态环境、systemd 托管主进程”的结构可以工作，但容器里的配置随镜像发布，升级配置时必须重新构建或重建容器。上面的示例增加了只读配置和 Bark Key 挂载，便于轮换密钥和备份配置。

### 7.2 安装启动前检查

创建 `/usr/local/sbin/ast-test-security-check`：

```bash
cat > /usr/local/sbin/ast-test-security-check <<'EOF'
#!/bin/bash
set -euo pipefail
spec=$(docker inspect ast-test)

jq -e '.[0] |
  .Config.Image == "asterisk-quectel:armv7" and
  .HostConfig.Privileged == false and
  .HostConfig.RestartPolicy.Name == "no" and
  .HostConfig.NetworkMode == "host" and
  ([.Mounts[] | select(.Destination == "/etc/asterisk" and .RW == false)] | length == 1) and
  ([.Mounts[] | select(.Destination == "/etc/box-provision/bark-key" and .RW == false)] | length == 1)
' <<< "$spec" >/dev/null

for port in 0 1 2 3 4; do
    dev="/dev/ttyUSB$port"
    test -c "$dev"
    jq -e --arg dev "$dev" 'any(.[0].HostConfig.Devices[];
      .PathOnHost == $dev and .PathInContainer == $dev)' <<< "$spec" >/dev/null
done

test -f /opt/asterisk/etc/quectel.conf
test -f /opt/asterisk/etc/extensions.conf
test -x /opt/asterisk/bin/bark-notify
test -s /etc/box-provision/bark-key
test "$(stat -c '%u:%g %a' /etc/box-provision/bark-key)" = "0:0 600"
unsafe=$(find /opt/asterisk/etc -xdev -perm /022 -print -quit)
test -z "$unsafe" || { echo "Unsafe Asterisk configuration permissions" >&2; exit 1; }
echo "Asterisk container configuration checks passed"
EOF
chmod 0755 /usr/local/sbin/ast-test-security-check
```

它检查镜像名称、特权模式、重启策略、设备映射与配置权限。镜像名称检查用于防止用错镜像，不代表验证了镜像签名或来源，也不能替代完整安全审计。

### 7.3 安装 Asterisk 自启动服务

创建 `/etc/systemd/system/ast-test-asterisk.service`：

```ini
[Unit]
Description=Asterisk with Quectel in Docker
Wants=docker.service
After=docker.service
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStartPre=/usr/local/sbin/quectel-usb-prepare.sh
ExecStartPre=/usr/local/sbin/ast-test-security-check
ExecStartPre=/bin/sh -c 'running=$$(/usr/bin/docker inspect -f "{{.State.Running}}" ast-test 2>/dev/null || true); if [ "$$running" != "true" ]; then /usr/bin/docker start ast-test; fi'
ExecStartPre=/usr/bin/docker exec ast-test /bin/sh -c 'for n in 0 1 2 3 4; do test -c /dev/ttyUSB$$n || exit 1; done'
ExecStart=/usr/bin/docker exec ast-test /usr/sbin/asterisk -f -U root -G root
ExecStop=-/usr/bin/docker exec ast-test /usr/sbin/asterisk -rx "core stop gracefully"
ExecStopPost=-/usr/bin/docker stop -t 10 ast-test
Restart=always
RestartSec=15
TimeoutStartSec=90
TimeoutStopSec=30
KillMode=process

[Install]
WantedBy=multi-user.target
```

这份参考配置的顺序是：

```text
加载驱动 → 每次写入 new_id → 等待并核对 ttyUSB
        → 检查容器配置 → docker start → 核对容器内串口 → Asterisk 前台运行
```

`StartLimitIntervalSec=0` 应写在 `[Unit]` 中。单元文件里的 `$$n` 用来把 `$n` 传给内部 shell，避免被 systemd 当作环境变量处理。启动前脚本返回失败时，后续 `docker start` 不执行；服务等待 15 秒后重试。USB 模块迟到时只影响这个业务服务，无需阻塞整个 Docker 服务。

`asterisk -f` 保持前台运行，让 systemd 通过 `docker exec` 观察进程退出。`ExecStopPost` 在服务停止后关闭业务容器，使下次启动重新设置设备映射；`Restart=always` 会恢复正常或异常退出的进程，但管理员执行 `systemctl stop` 时不会立即重启。

参考单元会先执行 USB 准备和安全检查，再按需启动 `ast-test`，随后运行 Asterisk。已有设备应通过 `systemctl cat ast-test-asterisk.service` 核对实际顺序；服务日志应能看到安全检查通过。若容器的 Docker 重启策略仍是 `always`，Docker 守护进程重启时可能绕过 systemd 前置脚本。需要严格保证“每次容器启动前写入 `new_id`”时，应把容器策略设为 `no`，只让 systemd 管理容器生命周期。

启用服务：

```bash
systemctl daemon-reload
systemctl enable --now ast-test-asterisk.service
systemctl status ast-test-asterisk.service --no-pager -l
```

今后统一使用：

```bash
systemctl restart ast-test-asterisk.service
```

**直接运行 `docker start ast-test` 或 `docker restart ast-test` 会绕过这些前置步骤。** Docker 没有自动调用该 systemd 单元的机制；“每次先 echo”的保证依赖于统一使用此入口，并关闭 Docker 自身的自动重启策略。

### 7.4 验证业务，而不仅是容器状态

```bash
docker exec ast-test ls -l /dev/ttyUSB0 /dev/ttyUSB1 /dev/ttyUSB2 /dev/ttyUSB3 /dev/ttyUSB4
docker exec ast-test asterisk -rx 'core show uptime'
docker exec ast-test asterisk -rx 'module show like chan_quectel'
docker exec ast-test asterisk -rx 'quectel show devices'
docker exec ast-test asterisk -rx 'dialplan show from-quectel'
journalctl -u ast-test-asterisk.service -n 80 --no-pager
ss -lntup
```

`docker ps` 中显示 `Up` 只说明本例的 `sleep infinity` 还在运行，不能证明 Asterisk 或蜂窝模块健康。确认模块正常注册后，向 SIM 卡发送一条测试短信，检查 Bark 收到的内容。


## 8. 可选：网页配置 Wi-Fi，联网后通知 IP

### 8.1 单网卡切换的限制

这台盒子的 Wi-Fi 同时承担热点和连接路由器的任务。常见旧驱动无法稳定地同时进行 AP 与 STA 工作，热点运行期间的扫描缓存也可能不完整。

因此建议采用如下流程：

1. 开机先尝试已保存的普通 Wi-Fi；有配置时等待一个有限时间。
2. 没有可用连接时，开启 `Box-Setup` 热点。
3. 手机或电脑连上热点，访问 `http://192.168.50.1`。
4. 网页先确认收到配网请求，再由后台关闭热点、切换到目标 Wi-Fi。
5. 成功后由 Bark 通知地址；失败则恢复热点。
6. 电脑切换到目标 Wi-Fi 后，使用通知中的 IP 或 `.local` 主机名 SSH。

热点关闭时浏览器会断线，所以不能保证它还能收到新网络上的 IP。让页面一直轮询并不能消除这种链路中断。没有外网时，Bark 也无法送达，需要通过路由器 DHCP 租约或网线找回设备。

### 8.2 创建带密码的配网热点

先确认网卡支持 AP 模式，网卡名称不是 `wlan0` 时同步修改后续配置。当前设备使用 Realtek USB 无线网卡和 `wlan0`，NetworkManager 中的 `Box-Setup` 是 `192.168.50.1/24` 的 shared AP 配置，默认没有自动连接。

```bash
apt install -y iw
iw list
nmcli device status
```

在 `Supported interface modes` 中寻找 `AP`。以下配置固定使用 2.4 GHz，并禁用热点自动抢占普通 Wi-Fi：

```bash
nmcli connection add type wifi ifname wlan0 con-name Box-Setup ssid Box-Setup
nmcli connection modify Box-Setup \
  802-11-wireless.mode ap 802-11-wireless.band bg \
  ipv4.method shared ipv4.addresses 192.168.50.1/24 \
  ipv6.method disabled connection.autoconnect no
read -r -s -p '设置 8–63 个 ASCII 字符的热点密码: ' SETUP_PSK
printf '\n'
nmcli connection modify Box-Setup wifi-sec.key-mgmt wpa-psk wifi-sec.psk "$SETUP_PSK"
unset SETUP_PSK
```

如果已有同名热点，修改现有连接即可，不要重复创建。NetworkManager 的 shared 模式负责 DHCP 和共享配置；部分发行版还需要安装 `dnsmasq-base`，不要再启一个占用相同端口的独立 DHCP 服务。开放热点只适合第一次救援，长期运行应设置 WPA2-PSK，并在页面中说明热点密码。

配置热点之前先检查本机已有的 Web 站点。一个 Nginx 实例只能由一个 default server 接收同一端口的默认请求；不要让已有的 PHP、WebDAV 或公网域名站点意外接管配网页面。配网盒子只需要一个限制来源网段的 server 块，其他站点应停用或明确绑定到不同端口。

配网页面提交的是目标 Wi-Fi 密码，因此热点本身也应使用 WPA2-PSK，并且只允许自己的终端接入。

### 8.3 安装独立的配网服务

下面是依据实际流程整理的简化版本，仅支持开放网络和 WPA/WPA2 个人网络。它直接创建目标 Wi-Fi 配置，再激活连接，避免把“当前扫描列表中看不到 SSID”作为拒绝连接的前置条件。隐藏 SSID 会启用主动探测；WPA 企业认证、强制 WPA3 网络不在此示例范围内。

它使用临时配置名完成验证，成功后替换自己上次保存的配置；不会清空其他 Wi-Fi 配置。热点开启期间没有自动刷新网络列表，直接输入 SSID 即可。

创建 `/usr/local/libexec/box-provision.py`：

```bash
install -d -m 0755 /usr/local/libexec
cat > /usr/local/libexec/box-provision.py <<'EOF'
#!/usr/bin/python3
import json
import os
import subprocess
import threading
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

DEVICE = "wlan0"
AP = "Box-Setup"
ACTIVE = "box-managed-wifi"
PENDING = "box-pending-wifi"
busy = threading.Lock()

def run(*args, timeout=20):
    result = subprocess.run(args, stdout=subprocess.PIPE, stderr=subprocess.DEVNULL,
                            text=True, timeout=timeout, env={**os.environ, "LC_ALL": "C"})
    if result.returncode:
        raise RuntimeError("command failed")
    return result.stdout.strip()

def best_effort(*args):
    try:
        run(*args)
    except (RuntimeError, OSError, subprocess.TimeoutExpired):
        pass

def profile():
    return run("nmcli", "-g", "GENERAL.CONNECTION", "device", "show", DEVICE)

def current_connection():
    state = run("nmcli", "-g", "GENERAL.STATE", "device", "show", DEVICE)
    name = profile()
    if not state.startswith("100") or name in (AP, "", "--"):
        return None
    addresses = run("nmcli", "-g", "IP4.ADDRESS", "device", "show", DEVICE)
    for line in addresses.splitlines():
        address = line.split("/", 1)[0]
        if address and address != "192.168.50.1":
            return name, address
    return None

def connect(ssid, password):
    try:
        time.sleep(1)  # Allow the HTTP 202 response to leave the interface.
        best_effort("nmcli", "connection", "delete", PENDING)
        run("nmcli", "connection", "add", "type", "wifi", "ifname", DEVICE,
            "con-name", PENDING, "ssid", ssid,
            "connection.autoconnect", "no", "802-11-wireless.hidden", "yes")
        if password:
            run("nmcli", "connection", "modify", PENDING,
                "wifi-sec.key-mgmt", "wpa-psk", "wifi-sec.psk", password)
        run("nmcli", "connection", "modify", PENDING, "ipv4.method", "auto",
            "ipv4.route-metric", "50", "connection.autoconnect-priority", "10")
        best_effort("nmcli", "connection", "down", AP)
        best_effort("nmcli", "device", "wifi", "rescan", "ifname", DEVICE)
        run("nmcli", "--wait", "45", "connection", "up", PENDING,
            "ifname", DEVICE, timeout=55)
        result = current_connection()
        if not result or result[0] != PENDING:
            raise RuntimeError("no IPv4 address")
        best_effort("nmcli", "connection", "delete", ACTIVE)
        run("nmcli", "connection", "modify", PENDING, "connection.id", ACTIVE,
            "connection.autoconnect", "yes")
        print("Wi-Fi connected; network notification will follow", flush=True)
    except (RuntimeError, OSError, subprocess.TimeoutExpired):
        best_effort("nmcli", "connection", "delete", PENDING)
        best_effort("nmcli", "connection", "up", AP)
        print("Wi-Fi setup failed; restoring setup hotspot", flush=True)
    finally:
        busy.release()

PAGE = """<!doctype html><html lang="zh-CN"><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>盒子配网</title><style>body{max-width:32rem;margin:3rem auto;padding:1rem;
font:16px system-ui}input,button{box-sizing:border-box;width:100%;padding:.8rem;
margin:.5rem 0}p{line-height:1.6}</style><h1>连接 Wi-Fi</h1>
<p>提交后热点会暂时断开。成功后通过 Bark 获取地址；失败后重新连接 Box-Setup。</p>
<form id="form"><label>Wi-Fi 名称<input id="ssid" required></label>
<label>密码（开放网络留空）<input id="password" type="password"></label>
<button id="submit">连接</button></form><p id="message"></p>
<script>
document.getElementById('form').onsubmit=async(event)=>{
event.preventDefault();const button=document.getElementById('submit');
const message=document.getElementById('message');button.disabled=true;
try{const response=await fetch('/api/connect',{method:'POST',
headers:{'Content-Type':'application/json'},body:JSON.stringify({
ssid:document.getElementById('ssid').value,password:document.getElementById('password').value})});
const result=await response.json();message.textContent=result.message;
if(!response.ok)button.disabled=false;
}catch(error){message.textContent='连接已中断，请等待 Bark 通知或重新连接配网热点。';}
};</script></html>"""

class Handler(BaseHTTPRequestHandler):
    def log_message(self, *args):
        pass

    def reply(self, code, body, content_type="application/json; charset=utf-8"):
        data = body.encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(data)))
        self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(data)

    def do_GET(self):
        self.reply(200, PAGE, "text/html; charset=utf-8")

    def do_POST(self):
        if self.path != "/api/connect":
            self.reply(404, '{"message":"Not found"}')
            return
        if self.headers.get("Content-Type", "").split(";")[0] != "application/json":
            self.reply(415, '{"message":"JSON required"}')
            return
        if self.headers.get("Origin") not in (None, "http://192.168.50.1"):
            self.reply(403, '{"message":"Invalid origin"}')
            return
        try:
            length = int(self.headers.get("Content-Length", "0"))
            if not 0 < length <= 4096:
                raise ValueError()
            payload = json.loads(self.rfile.read(length))
            ssid, password = payload["ssid"], payload.get("password", "")
            if not isinstance(ssid, str) or not 1 <= len(ssid.encode("utf-8")) <= 32:
                raise ValueError()
            if not isinstance(password, str) or len(password) > 64 or "\x00" in ssid + password:
                raise ValueError()
        except (ValueError, KeyError, TypeError):
            self.reply(400, '{"message":"SSID 或密码格式无效"}')
            return
        if not busy.acquire(blocking=False):
            self.reply(409, '{"message":"已有配网任务正在执行"}')
            return
        try:
            if profile() != AP:
                busy.release()
                self.reply(409, '{"message":"当前不处于配网热点模式"}')
                return
        except (RuntimeError, OSError, subprocess.TimeoutExpired):
            busy.release()
            self.reply(503, '{"message":"无线接口尚未就绪"}')
            return
        threading.Thread(target=connect, args=(ssid, password), daemon=True).start()
        self.reply(202, '{"message":"正在切换网络，请等待 Bark 通知"}')

def monitor():
    offline_since = time.monotonic()
    last_notified = None
    retry_at = 0
    while True:
        if busy.acquire(blocking=False):
            try:
                current = current_connection()
                if current:
                    offline_since = time.monotonic()
                    if current != last_notified and time.monotonic() >= retry_at:
                        name, address = current
                        hostname = os.uname().nodename
                        body = f"{name}\nIP: {address}\nSSH: ssh root@{address}\n主机名: {hostname}.local"
                        try:
                            run("/usr/local/bin/bark-notify", "盒子已联网", body, "box", timeout=45)
                            last_notified = current
                            print("Network notification accepted by Bark", flush=True)
                        except (RuntimeError, OSError, subprocess.TimeoutExpired):
                            retry_at = time.monotonic() + 60
                            print("Network notification failed; will retry", flush=True)
                else:
                    last_notified = None
                    if profile() != AP and time.monotonic() - offline_since >= 30:
                        best_effort("nmcli", "connection", "up", AP)
            except (RuntimeError, OSError, subprocess.TimeoutExpired):
                print("Waiting for NetworkManager or Wi-Fi interface", flush=True)
            finally:
                busy.release()
        time.sleep(5)

threading.Thread(target=monitor, daemon=True).start()
ThreadingHTTPServer(("127.0.0.1", 8765), Handler).serve_forever()
EOF
chmod 0755 /usr/local/libexec/box-provision.py
```

这里将“联网后通知”放在独立监控循环中，因此网页配网和开机自动连接都能触发。只有 Bark 确认接受后才记录本次已通知；发送失败会稍后重试，避免 DHCP 已完成但 DNS 或外网尚未就绪时漏掉通知。同一次稳定连接不重复通知，检测到断线后再连接会再次通知；短于轮询间隔的断线可能被略过。

部署后应按验收清单做一次冷启动和断网测试。若无线驱动在 30 秒内还没准备好，需要根据 `journalctl` 调整等待策略；热点激活后也不会后台持续抢回普通 Wi-Fi，以免正在配网时断线。

### 8.4 使用 Nginx 限制配网页面的访问范围

后端只监听 `127.0.0.1:8765`，由 Nginx 提供网页。在专门用于配网、没有其他网站的盒子上，新建 `/etc/nginx/sites-available/box-provision`：

```nginx
server {
    listen 80;
    server_name 192.168.50.1;
    client_max_body_size 4k;

    location / {
        allow 192.168.50.0/24;
        allow 127.0.0.1;
        deny all;
        proxy_pass http://127.0.0.1:8765;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 15s;
    }
}
```

```bash
ln -sfn /etc/nginx/sites-available/box-provision /etc/nginx/sites-enabled/box-provision
nginx -t && systemctl reload nginx
systemctl enable nginx
```

如果现有 Nginx 已有同名虚拟主机或相同用途的配网配置，应修改现有文件。限制来源网段是访问边界的一部分，不是用户身份认证；不要为这个配网页面配置公网端口转发。

创建 `/etc/systemd/system/box-provision.service`：

```ini
[Unit]
Description=Wi-Fi setup hotspot and network notification
Wants=NetworkManager.service nginx.service
After=NetworkManager.service nginx.service

[Service]
Type=simple
Environment=LC_ALL=C
ExecStart=/usr/bin/python3 /usr/local/libexec/box-provision.py
Restart=always
RestartSec=5
User=root
UMask=0077

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now box-provision.service
journalctl -u box-provision.service -n 50 --no-pager
```

用 root 运行是为了与旧系统的 NetworkManager 权限兼容。更严格的部署可以为专用账户配置最小 Polkit 权限。

### 8.5 配置一个便于记忆的主机名

```bash
hostnamectl set-hostname sms-box
```

在 `/etc/avahi/avahi-daemon.conf` 已有的 `[server]` 节中设置：

```ini
allow-interfaces=wlan0,eth0
use-ipv6=no
```

```bash
systemctl restart avahi-daemon
```

电脑与盒子接入同一局域网后，可以尝试 `ssh root@sms-box.local`。`.local` 依赖 mDNS，校园网隔离、跨 VLAN 或终端不支持 mDNS 时可能失败；此时使用实际 IP。


## 9. 部署后的验收

完成配置后，至少验证这些场景：

- 已保存 Wi-Fi 时开机自动连接，收到带地址的 Bark 通知。
- 目标 Wi-Fi 不可用时出现配网热点，浏览器可访问 `192.168.50.1`。
- 配网密码输入错误时，热点能够恢复。
- 模块插入并完成 USB 枚举后，宿主机出现预期的字符设备。
- 通过 `systemctl restart ast-test-asterisk.service` 重启时，日志先出现 USB 准备成功，再出现容器和 Asterisk 启动结果。
- 模块缺失时业务启动失败并重试，SSH 与配网服务仍然可用。
- 容器中五个串口可见，Asterisk CLI 可响应，实际测试短信能够推送。
- 重启设备后重新验证上述业务；单次服务重启成功不能代替冷启动测试。

在目标环境中，USB 准备脚本、systemd 前置步骤、五个字符设备映射、Asterisk 20.6.0、`chan_quectel` 注册和 Bark 短信推送均已完成过端到端验证。镜像构建、密钥轮换和配网热点仍应在自己的硬件上做冷启动验收。

## 参考资料

- [Asterisk chan_quectel 项目](https://github.com/IchthysMaranatha/asterisk-chan-quectel)
- [HiNAS 固件下载](https://www.ecoo.top/download)
- [HiNAS 机顶盒刷机教程](https://www.ecoo.top/docs/category/%E6%9C%BA%E9%A1%B6%E7%9B%92%E5%88%B7%E6%9C%BA%E6%95%99%E7%A8%8B)
- [Wi-Fi 安装脚本仓库](https://gitee.com/xjxjin/scripts)
- [本文参考的 chan_quectel 仓库](https://github.com/IchthysMaranatha/asterisk-chan-quectel)
- [chan_quectel 配置示例](https://github.com/IchthysMaranatha/asterisk-chan-quectel/blob/3d45c7f072131296a7e3c1a4faf5bb18751dbd87/etc/quectel.conf)
- [Bark 服务端与 API 说明](https://github.com/Finb/Bark)

