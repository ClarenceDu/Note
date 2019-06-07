# Git 查看某次commit的内容 #
## Git show ##
知道commit id的情况下:

1. 获取commit id

  	` git log `

2. 查看commit内容

 	`git show commit_id`
	
3. 查看最近n次提交的修改

   	`git log -p -n`

	指定n为1则可以查看最近一次修改的内容