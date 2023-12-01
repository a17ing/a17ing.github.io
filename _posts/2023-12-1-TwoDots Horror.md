---
title: TwoDots Horror
categories:
  - CTF
tags:
  - XSS，
  - CSP绕过
---
## 0x00 前言
[TwoDots Horror](https://app.hackthebox.com/challenges/twodots-horror)是一道有关XSS，和CSP绕过的题。黑盒的话也能做，只是要全靠猜。白盒的话，做的更快。这道题非常有趣，在[Bookworm](https://app.hackthebox.com/machines/Bookworm)这台机器上的User部分也用到了大概这种思路。

---
## 0x01 黑盒思路
![1.png](/assets/img/2023-12-1-TwoDots-Horror/1.png)
映入眼帘的是一个登录与注册页面，注册一个账号进去后发现有两个功能。
1. 输入一段话，必须有两个`.`，而且需要管理员审核，就说明管理员会看到我们输入的东西。
2. 可以上传一个头像。
![2.png](/assets/img/2023-12-1-TwoDots-Horror/2.png)
![3.png](/assets/img/2023-12-1-TwoDots-Horror/3.png)
思路：输入js代码盗取管理员cookie，但是js代码输入有限制，只能通过加载图片，来执行任意js代码。

---
