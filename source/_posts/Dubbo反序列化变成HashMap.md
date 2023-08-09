---
title: Dubbo反序列化变成HashMap
date: 2023-08-09 17:38:09
tags:
  - Dubbo
  - Hession
  - HashMap
  - Java
  - DENY_CLASS
---

### 前言

前一阵遇到一个很有意思的问题，Dubbo客户端调用服务端接口，报错了，抛了一个异常，结果到了客户端这边，却变成了一个HashMap，而HashMap里面存放的正是报错信息！

当时的第一反应就是不可能，这不可能。

### 排查问题

虽然嘴上说不可能，但是事实就摆在眼前，没办法，也只能接受这个现实。那接下来就是漫长的排查问题了（其实也还好，就一个多小时）。

经过漫长的排查，最终定位到了Hessian中的ClassFactory这个类。ClassFactory这个类看名字就知道干什么的了，负责类的加载。当我们调用Dubbo接口返回结果时，就是通过这个类来加载类信息的。

我们可以看下它的加载代码：

```java
public Class<?> load(String className) throws ClassNotFoundException
{
    if (isAllow(className)) {
        return Class.forName(className, false, _loader);
    }
    else {
        log.log(Level.SEVERE, className + " in blacklist or not in whitelist, deserialization  with type 'HashMap' instead.");
        return HashMap.class;
    }
}
```

从代码上，我们就能看出来，当指定的类不被允许时，就会返回HashMap类信息。

但是为什么我们的类不被允许加载呢？？？

继续看

```java
private boolean isAllow(String className)
{
    ArrayList<Allow> allowList = _allowList;
    LinkedList<Allow> allowList = _allowList;

    if (allowList == null) {
        return true;
    }

    if (_allowClassSet.containsKey(className)) {
        return true;
    }

    int size = allowList.size();
    for (int i = 0; i < size; i++) {
        Allow allow = allowList.get(i);

        Boolean isAllow = allow.allow(className);

        if (isAllow != null) {
            if (isAllow) {
                _allowClassSet.put(className, className);
            }
            return isAllow;
        }
    }
    if (_isWhitelist) {
        return false;
    }

    _allowClassSet.put(className, className);
    return true;
}

static {
    _staticAllowList = new ArrayList<Allow>();

    _staticAllowList.add(new Allow("java\\..+", true));
    ClassLoader classLoader = ClassFactory.class.getClassLoader();
    try {
      	// 看这里，看这里
        String[] denyClasses = readLines(classLoader.getResourceAsStream("DENY_CLASS"));
        for (String denyClass : denyClasses) {
            if (denyClass.startsWith("#")) {
                continue;
            }
            _staticAllowList.add(new AllowPrefix(denyClass, false));
        }
    } catch (IOException ignore) {

    }
}
```

通过上述代码，我们可以看到，Hessian在内部维护了一份DENY_CLASS名单。如果你的结果类在这个名单里面或者父类在这个名单里面(通过类路径匹配)，抱歉，你只能获得一份HashMap。

解决办法也很简单，避开这份名单就好。



> 参考: [Add Default Deny List](https://github.com/apache/dubbo-hessian-lite/commit/8d49db4128aea8fc6e469a25e19b4d50af91e0bd)
>
> 参考: [CVE-2022-39198 Apache Dubbo Hession Deserialization分析](https://blog.csdn.net/include_voidmain/article/details/128476795)
