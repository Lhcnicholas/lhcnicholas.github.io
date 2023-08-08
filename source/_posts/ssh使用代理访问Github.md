---
title: ssh使用代理访问Github
date: 2023-05-10 15:05:20
tags: 
  - Git
  - Github
---

平时在GitHub上看到一些有意思的开源项目，或者自己的代码，想拉下来，结果因为种种原因很难拉下来，这时候就需要用到代理。

### HTTP协议拉取代码

如果使用http协议拉取代码还好说，一行代码就搞定了

```bash
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
```

或者直接把这串配到`.zshrc`里面去，都可以，这样就不需要每次都敲一遍了。

### SSH协议拉取代码

但是呢，使用使用http协议拉取代码总要输入密码，就很麻烦，这时候SSH就登场了。

SSH设置代理的方式稍微复杂一点，不过也就一点点🤏。

1. 我们要在`～/.ssh`目录下创建一个`config`文件，名字就叫config，无后缀。

2. 在`config`文件里面配置如下：

   ```sh
   Host github.com
     User git
     HostName ssh.github.com
     Port 443
     ProxyCommand nc -v -x 127.0.0.1:7890 %h %p
     IdentitiesOnly yes
     IdentityFile ~/.ssh/github_rsa
   ```

3. 配置好之后我们可以用` ssh -T git@github.com` 这个命令来测试一下，正常情况下会有如下响应。

```sh
 ~ ssh -T git@github.com
Connection to ssh.github.com port 443 [tcp/https] succeeded!
Hi Lhcnicholas! You've successfully authenticated, but GitHub does not provide shell access.
```

#### 配置说明

`User git` ：这个是固定的

`HostName ssh.github.com`：在使用443端口情况下主机名为`ssh.github.com`

`Port 443`: 22端口有时会因为各种原因连不上，所以我们用443端口，稳一些

`ProxyCommand nc -v -x 127.0.0.1:7890 %h %p`: 中间是你的代理地址，具体命令每一个参数什么意思，不用太在意（主要我自己也说不太清）

`IdentitiesOnly yes`: 是否只用指定的标识文件来验证身份，不再尝试其他方式（可有可无，影响不大）

`IdentityFile ~/.ssh/github_rsa`: 你的标识文件的目录地址（私钥地址，不是公钥）。

> 参考：[GitHub 加速终极教程(大佬)](https://hellodk.cn/post/975)
>
> 参考：[在HTTPS端口使用SSH](https://docs.github.com/zh/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)

