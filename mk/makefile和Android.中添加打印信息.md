我们常常需要在makefile/android.mk文件中添加打印信息来显示某个变量的值，或者用来控制makefile/android.mk的执行过程。makefile/android.mk文件都遵循gnu make的语法规则，查看gun make手册可知，gnu make提供了两个函数用来输出打印信息或者控制make的执行过程，分别是：

$(error TEXT......)

这个函数被执行的时候，会输出：TEXT......，并且终止make的执行。

$(warning TEXT......)

这个函数被执行的时候，会输出：TEXT......，但是make会继续执行下去。

其中“TEXT.....”可以替换为对变量的取值来输出变量的信息，例如：$(warning $(VAR))，那么该函数执行的时候会输出变量VAR的值。