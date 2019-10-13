# 编写mk文件时遇到的问题，防止以后再被坑 #
## 1.当是给某个变量赋值后，最好在其后面打印这个变量，看看设置值是否和自己想的一样，使用$(warning $(value))打印这个变量值是否正确。 ##
踩雷：<br>
之前写中间件框架时，一个aidl调用另外一个aidl，import一直报找不到要调用的aild错误，但mk文件中LOCAL_AIDL_INCLUDES中加了要调用的aild的绝对路径。最后加个打印LOCAL_AIDL_INCLUDES发现是绝对路径值写错了。