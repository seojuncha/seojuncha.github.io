---
title: "Buildroot와 QEMU를 활용한 cortex-a7 시뮬레이션"
date: 2025-02-26
tags: [Buildroot, QEMU, ARM Cortex-A7]
description: ""
draft: true
---

## Buildroot 다운로드

## Build
```shell
sudo apt-get install libncurses-dev
```

```shell
$ make menuconfig
```


Filesysystem images
ext2/3/4 root filesystem
Select ext4

tar the root filesystem
Compression method -> gzip

U-boot config
qemu_arm_vexpress_defconfig


gstreamer-1.x
gst1-plugins-good
gst1-rtsp-server
****
nginx
ngx_http_ssl_module
ngx_http_auth_digest
