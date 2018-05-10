---
title: NextCloud 网盘
toc: true
tags:
  - NextCloud
date: 2009-01-01 21:40:52
updated: 2009-01-01 21:40:52
categories:
  - software
---
# NextCloud 网盘

> Nextcloud 是一套用于创建网络硬盘的客户端－服务器软件。其功能与 Dropbox 相近，但 Nextcloud 是自由及开放源代码软件，每个人都可以在私人服务器上安装并运行它。

- Nextcloud [官网](https://nextcloud.com/)
- [Nextcloud](https://github.com/nextcloud) Github Pinned repositories

## 介绍

Nextcloud 的前世今生

## 安装

### 部署 PHP 环境

- nextcloud 安装条件
  - PHP ≥ 5.6.0 (官方推荐 7.0 以上版本)
  - MySQL / SQLite (推荐使用 MySQL)
  - Apache / NGINX (本次环境部署采用 NGINX)
  - PHP-FPM (老版本 php-fastcgi 已被 PHP-FPM 代替 )

- 本次安装采用 lnmp 一键包; [传送门](https://lnmp.org/install.html)
  - 安装步骤[传送门](https://lnmp.org/install.html)

### 安装 Nextcloud 服务

### 上传

## 运维 & 管理

## 应用场景

### 私人网盘

### 局域网内分享

## 数据传输工具

### S3 协议

- [Minio](https://www.minio.io/)
  - [中文介绍](https://github.com/minio/minio/blob/master/README_zh_CN.md)
  - [](http://blog.just4fun.site/install-Minio-Cloud-Storage.html)

### P2P 场景

- [Resilio Sync](https://www.resilio.com/)
  - BitTorrent Sync
- [Syncthing](https://syncthing.net/)
  - [Syncthing](https://github.com/syncthing/syncthing) Github Code
  - Syncthing 和 BitTorrent Sync 完成的是同一件事情
  - Syncthing是一个开源的文件同步客户端与服务器软件，采用Go语言编写。它使用了其独有的对等自由块交换协议。