---
title: 利用Alfred Workflow获取Google Authenticator动态密码
date: 2023-05-09 13:23:47
categories:
    - Mac
tags: 
    - Mac
    - Alfred
    - Worflow
    - Google Authenticator
---

公司的跳板机登陆需要用到Google Authenticator，每次都要掏出手机，找到App，对着密码一个一个数字敲，就很麻烦，就想着，就不能所有操作都在电脑上吗？功夫不负有心人，经过一番Google，终于找到了一个办法。

原来Google Authenticator使用的是公开的TOTP(Time-based One-Time Password)算法，而且现在已经有开源项目可以提取密钥了，那事情就简单了。

## 办法
1. 手机上通过Google Authenticator的导出功能，导出密钥对应的二维码。

2. 使用相机或者微信去识别这个二维码，会得到一串代码，类似：otpauth-migration://offline?data=...

3. 使用[ExtractOtpSecrets](https://github.com/scito/extract_otp_secrets)来提取密钥

4. 将得到的密钥填入到workflow中。

   ![image-2023050917572027](https://lhc-img.oss-cn-hangzhou.aliyuncs.com/image-20230509175720274.png)

5. Alfred中输入`gga`获取你的动态密码吧。

![截屏2023-05-09 17.57.5](https://lhc-img.oss-cn-hangzhou.aliyuncs.com/%E6%88%AA%E5%B1%8F2023-05-09%2017.57.52.png)


附：Workflow下载地址: [下载地址](http://lhc-app.oss-cn-hangzhou.aliyuncs.com/GoogleAuth.alfredworkflow)


参考：https://zhuanlan.zhihu.com/p/362783435