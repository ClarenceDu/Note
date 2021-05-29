# git中文乱码

现象：

```shell
djm@djm-PC:~/Documents/Note$ git status ./
位于分支 master
您的分支与上游分支 'origin/master' 一致。

尚未暂存以备提交的变更：
  （使用 "git add/rm <文件>..." 更新要提交的内容）
  （使用 "git restore <文件>..." 丢弃工作区的改动）
	删除：     "\351\254\274\350\260\267\345\255\220/\346\234\254\347\273\217\351\230\264\347\254\246\344\270\203\346\234\257.md"

```

修复方式

```shell
djm@djm-PC:~/Documents/Note$ git config --global core.quotepath false
```

最终

```shell
djm@djm-PC:~/Documents/Note$ git status ./
位于分支 master
您的分支与上游分支 'origin/master' 一致。

尚未暂存以备提交的变更：
  （使用 "git add/rm <文件>..." 更新要提交的内容）
  （使用 "git restore <文件>..." 丢弃工作区的改动）
	删除：     鬼谷子/本经阴符七术.md

```

