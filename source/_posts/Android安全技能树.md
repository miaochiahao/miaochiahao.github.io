---
title: Android安全技能树
date: 2020-04-13 00:37:05
tags: Wiki
---

用于记录Android知识点，仅作为个人知识整理的大纲

## 漏洞类型

* 组件暴露
  * Activity
  * Service
  * Content-Provider
  * Receiver
* 权限设置不当
  * LaunchAnyWhere/BroadcastAnyWhere
  * 游离权限
  * 敏感组件使用Normal权限
  * sharedUserId
* 路径穿越
  * 下载
  * 保存
  * 解压
* 设计缺陷
  * 分屏场景
  * 静默安装能力与sdk-version版本
* WebView
  * 暴露JSBridge造成能力被恶意调用
* HTTPS
  * 证书不可信时继续访问造成中间人攻击
* DoS
  * NullPointerException
  * ClassCastException
  * IndexOutOfBoundsException
  * ClassNotFoundException
* SQL注入（Content-Provider）
* Logcat泄露敏感信息
* 使用/sdcard保存私有数据

## 挖掘技术

* Intent-Hook
* 流量监听
  * HttpCanary
* 脱壳技术
  * FART
  * Frida脱壳
* Hook技术
  * Xposed
  * Frida