# Git本地中文名文件显示问题 #
**现象：提交到git上的中文名文件上的中文变为显示数字**

   `.../\345\256\211\350\243\205MarkdownPad2.md`

解决方案：

    git config --global core.quotepath false

