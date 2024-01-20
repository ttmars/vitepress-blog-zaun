---
title: etcd-单实例部署
description: etcd-单实例部署
date: 2023-12-30
tags:
  - etcd
---

### 1. 二进制安装

截止到2023.12.30，etcd最新版本为v3.5.11

去GitHub上下载对应的二进制安装包，[https://github.com/etcd-io/etcd/releases](https://github.com/etcd-io/etcd/releases)，此处下载版本为：etcd-v3.5.11-linux-amd64.tar.gz

解压后启动服务，配置为可远程访问

```shell
./etcd  --advertise-client-urls='http://0.0.0.0:2379'  --listen-client-urls='http://0.0.0.0:2379'
```

### 2. 参考资料

- https://github.com/etcd-io/etcd/releases
- https://etcd.io/docs/v3.5/install/
- https://etcd.io/docs/v3.5/op-guide/configuration/

