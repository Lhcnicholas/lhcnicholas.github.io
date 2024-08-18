---
title: Git将不想提交又不想加入到.gitIgnore里面的文件忽略掉
date: 2024-08-18 11:47:00
tags:
  - git
---

在开发过程中，有时候会碰到一种情况，这个文件我不想提交，但是又不想把他加入到.gitignore文件里面去。

比如一些配置文件，我只是把地址配置到本地，完全不需要提交，这时候就很麻烦，每次提交的时候都要把它单独排除掉，有时候还容易误操作，就很烦。但又不能把它加入到.gitignore文件里面去，毕竟对于新增的配置，还是需要提交的。

这时候，我发现git有个命令：`update-index --skip-worktree` ,这个命令可以将你指定的文件忽略掉，使用方法也很简单：

```bash
# 假设文件名为 config.yaml
git update-index --skip-worktree config.yaml
```

如果你不想忽略了，也很简单：

```bash
git update-index --no-skip-worktree config.yaml
```



### 注意

这个方法虽好，但是毕竟没有git提示你这个文件修改了，时间长了，你可能都忘了这个文件修改了，这时候可以使用如下命令：

```bash
git ls-files -v | grep '^[S]'
```

`ls-files -v`这个命令会列出git目录下所有的文件，并显示出他们的状态，S开头 表示这个文件是被忽略掉的，被标记了`--skip-worktree`。



另外还有就是被忽略了的文件在你切换分支或者pull代码的时候会报错，因为你本地的文件和其他分支或远程上的文件状态不一致，这时候可以临时取消忽略就可以了。



### 解释

`git ls-files -v` 列出当前 Git 仓库中的所有文件，并显示它们的状态。

文件名前的标志：

- `H`：表示文件正常被跟踪。
- `h`：表示文件被标记为 `--assume-unchanged`。
- `S`：表示文件被标记为 `--skip-worktree`。
