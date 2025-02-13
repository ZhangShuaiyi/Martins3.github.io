# 基本设计

将参数分离开，组合测试

- corn list
- corn setup
- corn execute # 使用 ansible 来自动控制
- corn avocado # 使用 avocado 来进行测试

## 自动安装

### [ ] Ubuntu 自动安装
- [ ] 安装完成之后，不知道为什么，无法自动重启，而是一直在等待网络。
- [ ] 安装之后，可以替换 kernel 吗？

### [ ] Centos 自动安装


## 编译 ubuntu 内核 : 没太大必要，因为编译 rpm 也非常快
```txt
GEN     debian
Using default distribution of 'unstable' in the changelog
Install lsb-release or set $KDEB_CHANGELOG_DIST explicitly
dpkg-buildpackage -r"fakeroot -u" -a$(cat debian/arch)  -b -nc -uc
dpkg-buildpackage: info: source package linux-upstream
dpkg-buildpackage: info: source version 6.3.0-rc1-00002-g8ca09d5fa354-dirty-169
dpkg-buildpackage: info: source distribution unstable
dpkg-buildpackage: info: source changed by martins3 <martins3@nixos>
dpkg-architecture: warning: specified GNU system type x86_64-linux-gnu does not match CC system type x86_64-unknown-linux-gnu, try setting a correct CC environment variable
dpkg-buildpackage: info: host architecture amd64
 dpkg-source --before-build .
dpkg-source: info: using options from linux/debian/source/local-options: --diff-ignore --extend-diff-ignore=.*
dpkg-checkbuilddeps: error: cannot open /var/lib/dpkg/status: No such file or directory
dpkg-buildpackage: warning: build dependencies/conflicts unsatisfied; aborting
dpkg-buildpackage: warning: (Use -d flag to override.)
make[1]: *** [scripts/Makefile.package:127: bindeb-pkg] Error 3
make: *** [Makefile:1657: bindeb-pkg] Error 2
```
借用 https://github.com/a13xp0p0v/kernel-build-containers 的实现。

https://wiki.debian.org/BuildADebianKernelPackage
参考:
 make -j`nproc` bindeb-pkg

不知道为什么，还是这个错误:
```txt
  SYNC    include/config/auto.conf.cmd
make[2]: scripts/kconfig/conf: No such file or directory
make[2]: *** [scripts/kconfig/Makefile:77: syncconfig] Error 127
make[1]: *** [Makefile:693: syncconfig] Error 2
make: *** [Makefile:794: include/config/auto.conf.cmd] Error 2
```

## [x] build docker 镜像还是存在问题的
sudo apt install dpkg-dev rsync kmod cpio

## [x] 验证一下，是否可以用安装

.rw-r--r-- 8.9M root  9 Mar 17:08  linux-headers-6.3.0-rc1-00002-g8ca09d5fa354_6.3.0-rc1-00002-g8ca09d5fa354-4_amd64.deb
.rw-r--r--  13M root  9 Mar 17:08  linux-image-6.3.0-rc1-00002-g8ca09d5fa354_6.3.0-rc1-00002-g8ca09d5fa354-4_amd64.deb
.rw-r--r-- 1.3M root  9 Mar 17:08  linux-libc-dev_6.3.0-rc1-00002-g8ca09d5fa354-4_amd64.deb

虽然没有 debuginfo 包，但是我已经持有了 vmlinux ，其实无所谓

### 实际上，我们发现，只要是
1. make defconfig kvm_guest.config 几乎任何内核都可以拉起来的
2. 就是编译的内核有点大，不知道为什么。

## 需求
### 使用 grubby 自动切换内核，修改 kernel cmdline
```sh
sudo grubby --update-kernel=ALL --args="nokaslr console=ttyS0,9600 earlyprink=serial"
```
### 删除 guest 密码

### guest 中 oh-my-zsh 的基本命令

## 自动启动嵌套虚拟化
```c
function share() {
	cat <<'EOF' >/etc/systemd/system/nested.service
[Unit]
Description=nested virtualization setup

[Service]
Type=oneshot
ExecStart=/root/core/vn/docs/qemu/sh/alpine.sh

[Install]
WantedBy=getty.target
EOF

systemctl enable nested
}
```

## 增加 edk2 的环境搭建

## 使用 libvirt 也是可以装系统的，但是需要 ovs

## 打包方式
```txt
Kernel packaging:
  rpm-pkg             - Build both source and binary RPM kernel packages
  srcrpm-pkg          - Build only the source kernel RPM package
  binrpm-pkg          - Build only the binary kernel RPM package
```
