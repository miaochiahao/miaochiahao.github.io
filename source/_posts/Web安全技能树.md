---
title: Web安全技能树
date: 9999-04-13 00:36:36
tags: Wiki
---

用于记录Web知识点，仅作为个人知识整理的大纲

## 通用技能

### 前端

* XSS
  * Exploit
  * 绕过
* CSRF
* CORS使用不当
* JSONP
* XSSI
* CSS-Data-Exfil
* Post-Message引起数据泄露
* Click-Jacking
* 其他需要了解的概念
  * CSP
  * SOP
  * 
* ...

### 后端

* 命令注入
* SQL注入
* 文件上传/下载
* 横向/纵向越权
* HTTP-Request-Smuggling
* SSRF
* SSTI
* 反序列化
* ...

## 语言特性

### Java

* RMI/LDAP
* JNDI注入
* 反序列化
* SpringMVC
  * 无视后缀名匹配进行绕过（使用静态文件后缀名）
  * 视图注入
  * SpEL表达式注入
  * SpringMVC数据流（配置项、过滤器、拦截器、切面、）
* Struts OGNL表达式注入

### PHP

* 弱类型特性
* filter

### Python

* 反序列化

### Golang