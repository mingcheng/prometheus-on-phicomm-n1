# 在 Phicomm（斐讯）N1 上部署 Prometheus 和 Grafana

目前市场上出现了批矿难留下来的名为斐讯 N1 的机顶盒，有着不错的性能。对比树莓派 3B+，N1 有着更好的性价比以及外观。相对来说，N1 对比 树莓派3B+ 是更优的选择。

![N1](https://friable.rocks/_/2019_07_01/1561965398151224.jpg)

由于原先的 Prometheus 和 Grfana 是在台虚拟机里，从监控的角度叫上说单独的硬件更加合适些，所刚好入了台 N1 用来当作单独的监控系统。

## 硬件

购买的渠道先不说了，普遍价格依据成色从几十到一百出头不等。建议购买相对成色比较新带包装和三码的产品，由于底价低所以差价其实并不是很大。

下面硬件方面，斐讯的 N1 和 树莓派3B+ 做个对比：

| / |  斐讯 N1 | 树莓派 3B+ |
| ------ | ------ | ------ |
| CPU 和平台 | Amlogic ARM64 | BCM2835 ARMv7 |
| 内存 | 2G | 1G |
| 存储 | 自带 8g EMMC | 另外安装 SD 卡 |
| 外观 | 自带外壳 | 裸板，需要外壳自行购买 |
| 价格 | 100+ | 裸板200+，外壳和存储另算 |

具体性能方面的，[可以参考这篇文档](https://zocodev.com/phicomm-n1-box.html)这里不放具体的指标和数字了。

## 系统调整

目前二手市场上很多卖家都提供了刷机服务，我这边为了节省时间直接让卖家给刷了Armbian 系统。

到手以后 SSH 上去发现存储这块有些不对，还需要调整。因为原先还有部分 Android 的文件在其他的分区，可以直接执行 `blkid` 查看可以的块设备。

大概有那么几个块设备可以使用：

```
/dev/cache
/dev/tee
/dev/system
/dev/data
```

`/dev/data` 是目前的根分区，我们不用动它，而 `/dev/tee` 这个分区太小没有使用的价值，所以个人将 `/dev/system` 格式化为 ext4 mount 到了 /home 以及将 /dev/cache 作为 swap 分区备用（才 512MB 有点鸡肋）。

```
Filesystem      Size  Used Avail Use% Mounted on
udev            789M     0  789M   0% /dev
tmpfs           180M   17M  164M  10% /run
/dev/data       4.4G  2.4G  2.0G  56% /
tmpfs           900M     0  900M   0% /dev/shm
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           900M     0  900M   0% /sys/fs/cgroup
tmpfs           900M     0  900M   0% /tmp
/dev/system     1.2G  294M  855M  26% /home
log2ram          50M   13M   38M  25% /var/log
tmpfs           180M     0  180M   0% /run/user/1000
```

以及 free（已经运行了部分服务的情况）：

```
              total        used        free      shared  buff/cache   available
Mem:           1.8G        205M        837M         28M        756M        1.5G
Swap:          511M          0B        511M
```

所以，总体分区调整配置完了以后，大概是这样子总计占用的空间大概 7G 不到一点，但其实足够日常使用了（其实是不想在系统瘦身这块花更多的时间）。

调整完分区，然后关闭不必要的服务和启动项。关闭和删除红外线服务：

```
systemctl disable lircd.service lircd-setup.service lircd.socket lircd-uinput.service lircmd.service
apt remove -y lirc
```

关闭 NFS 服务，在集群里已经有 NFS 服务器了，所以不需要：

```
systemctl disable nfs-server
```

禁止图形界面启动的 Hook，这个其实没必要操作，但为了避免自启动有些图形应用：

```
systemctl disable graphical.target
```

后面添加清华的镜像源，安装 `Docker CE` 等操作就不复述了。这样子，重启以后系统层面的配置就完成了。

## 配置

下面主要说下这个机子安装 Prometheus 和 Grafana 遇到的些坑。首先，就是平台的问题，由于是 ARM64 的设备，所以直接用 Docker 镜像（默认 x86/amd64）是行不通的，需要使用针对平台的 Docker 镜像。

具体请参见 `docker-compose.yml` 文件的配置，注意镜像的名称。因为机子的原生可用容量较少，所以针对 Prometheus 最好追加个容量方面的限定参数，例如我这边配置了：

```yaml
- '--storage.tsdb.retention.time=6month'
- '--storage.tsdb.retention.size=2GB'
```

六个月或者总容量到达 2GB 的时候自动清除老的数据，默认 Prometheus 清除时间为 15d。

注意，`Alert Manager` 和 `Node Exporter` 以及 `Push GateWay` 都没有加入 Docker 的部署。

原因是，一来官方没有针对 ARM64 平台的镜像，二来这些服务相对比较简单、同时数据也不用纳入 Docker Volume 管理，因此就直接下载安装包运行。

## 后续

执行完 docker-compose 以后，就有了基础的 Prometheus 以及 Grafana 环境了，分别暴露的端口为 9090 以及 3000 。

Grafana 相关的 Dashboard 配置在 `./grafana` 目录中，如果有需要直接 import 即可。下面是效果图：

![Grafana](https://friable.rocks/_/2019_07_01/1561965398165370.png)

如有任何问题和疑问，欢迎探讨 mingcheng[at]outlook.com

`- eof -`
