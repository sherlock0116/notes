# 【Git】gitignore不生效

.gitignore规则不生效

## 1. 现象

不小心在 IDE 提示是否添加到 git 时，点了确定，发现管理了 .idea、target文件夹，

然后添加 .gitignore 文件夹，配置了以上规则，但是重新编译后，target 的修改在 git status 还是显示了修改！？

## 2. 原因

.gitignore 只能忽略那些原来没有被 track（之前没有 add 过）的文件，如果某些文件已经被纳入了版本管理中，则修改 .gitignore 是无效的。

## 3. 解决方案

解决方法就是先把本地缓存删除（改变成未track状态），然后再提交:

```sh
git rm -r --cached target
git rm -r --cached .idea
```

此后不再追踪track这两个文件夹
