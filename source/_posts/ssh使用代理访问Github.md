---
title: ssh使用代理访问Github
date: 2023-05-10 15:05:20
tags: 
  - Git
  - Github
---

平时在GitHub上看到一些有意思的开源项目，或者自己的代码，想拉下来，结果因为种种原因很难拉下来，这时候就需要用到代理。

如果使用http协议拉取代码还好说，一行代码就搞定了

```bash
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
```

或者直接把这串配到`.zshrc`里面去，都可以，这样就不需要每次都敲一遍了。



