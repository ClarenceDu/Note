# 作用
SELinux 开始为传统的自主访问控制 (DAC) 环境提供强制访问控制 (MAC) 保护功能。例如，软件通常情况下必须以 root 用户账号的身份运行，才能向原始块设备写入数据。在基于 DAC 的传统 Linux 环境中，如果根用户遭到入侵，攻击者便可以利用该用户身份向每个原始块存储设备写入数据。不过，可以使用 SELinux 为这些设备添加标签，以便被分配了 root 权限的进程可以只向相关政策中指定的设备写入数据。这样一来，该进程便无法覆盖特定原始块存储设备之外的数据和系统设置。
# 执行级别
- 宽容模式 - 仅记录但不强制执行 SELinux 安全政策。
- 强制模式 - 强制执行并记录安全政策。如果失败，则显示为 EPERM 错误。

# 关键文件
system/sepolicy目录中的文件 : 这些文件在编译后会包含 SELinux 内核安全政策，并涵盖上游 Android 操作系统。
通常情况下，您不能直接修改 
system/sepolicy 文件，但您可以添加或修改自己的设备专用政策文件（位于 /devicemanufacturer/device-name/sepolicy 目录中）
## 政策载体
Android 依靠 SELinux 的类型强制执行 (TE) 组件来实施其政策。以 *.te 结尾的文件是 SELinux 政策源代码文件，用于定义域及其标签。
## 政策的存放位置
在 Android 8.0 及更高版本中，政策位于 AOSP 中的以下位置：
- system/sepolicy/public。其中包括所导出的用于供应商特定政策的政策。所有内容都会纳入 Android 8.0 兼容性基础架构。公共政策会保留在不同版本上，因此您可以在自定义政策的 /public 中添加任何内容。正因如此，可存放在 /public 中的政策类型的限制性更强。将此目录视为相应平台的已导出政策 API：处理 /system 与 /vendor 之间的接口的所有内容都位于这里。
- system/sepolicy/private。包括系统映像正常运行所必需（但供应商映像政策应该不知道）的政策。
- system/sepolicy/vendor。包括位于 /vendor 但存在于核心平台树（非设备特定目录）中的组件的相关政策。这是构建系统区分设备和全局组件的软件工件；从概念上讲，这是下述设备专用政策的一部分。
- device/manufacturer/device-name/sepolicy。包含设备专用政策，以及对政策进行的设备自定义（在 Android 8.0 及更高版本中，该政策对应于供应商映像组件的相关政策）。
在 Android 11 及更高版本中，system_ext 和 product 分区还可以包含特定于分区的政策。system_ext 和 product 政策也分为公共政策和私有政策，且供应商可以使用 system_ext 和 product 的公共政策（例如系统政策）。
- SYSTEM_EXT_PUBLIC_SEPOLICY_DIRS。包括所导出的用于供应商特定政策的政策。已安装到 system_ext 分区。
- SYSTEM_EXT_PRIVATE_SEPOLICY_DIRS。包括 system_ext 映像正常运行所必需（但供应商映像政策应该不知道）的政策。已安装到 system_ext 分区。
- PRODUCT_PUBLIC_SEPOLICY_DIRS。包括所导出的用于供应商特定政策的政策。已安装到 product 分区。
- PRODUCT_PRIVATE_SEPOLICY_DIRS。包括 product 映像正常运行所必需（但供应商映像政策应该不知道）的政策。已安装到 product 分区。
## 格式
政策规则采用以下格式：allow source target:class permissions;，其中：
- source - 规则主题的类型（或属性）。谁正在请求访问权限？
- 目标 - 对象的类型（或属性）。对哪些内容提出了访问权限请求？
- 类 - 要访问的对象（例如，文件、套接字）的类型。
- 权限 - 要执行的操作（或一组操作，例如读取、写入）。

规则的一个示例如下：
```
allow untrusted_app app_data_file:file { read write };
```
这表示应用可以读取和写入带有 app_data_file 标签的文件。
上下文的描述文件
可以在上下文的描述文件中为你的对象指定标签。
- file_contexts 用于为文件分配标签，并且可供多种用户空间组件使用。在创建新政策时，请创建或更新该文件，以便为文件分配新标签。如需应用新的 file_contexts，请重新构建文件系统映像，或对要重新添加标签的文件运行 restorecon。在升级时，对 file_contexts 所做的更改会在升级过程中自动应用于系统和用户数据分区。此外，您还可以通过以下方式使这些更改在升级过程中自动应用于其他分区：在以允许读写的方式装载相应分区后，将 restorecon_recursive 调用添加到 init.board
- genfs_contexts 用于为不支持扩展属性的文件系统（例如 proc 或 vfat）分配标签。此配置会作为内核政策的一部分进行加载，但更改可能对内核 inode 无效。要全面应用更改，您需要重新启动设备，或卸载后重新装载文件系统。 此外，通过使用 context=mount 选项，您还可以为装载的特定系统文件（例如 vfat）分配特定标签。
- property_contexts 用于为 Android 系统属性分配标签，以便控制哪些进程可以设置这些属性。在启动期间，init 进程会读取此配置。
- service_contexts 用于为 Android binder 服务分配标签，以便控制哪些进程可以为相应服务添加（注册）和查找（查询）binder 引用。在启动期间，servicemanager 进程会读取此配置。
- seapp_contexts 用于为应用进程和 /data/data 目录分配标签。在每次应用启动时，zygote 进程都会读取此配置；在启动期间，installd 会读取此配置。
- mac_permissions.xml 用于根据应用签名和应用软件包名称（后者可选）为应用分配 seinfo 标记。随后，分配的 seinfo 标记可在 seapp_contexts 文件中用作密钥，以便为带有该 seinfo 标记的所有应用分配特定标签。在启动期间，system_server 会读取此配置。
- keystore2_key_contexts 用于为密钥库 2.0 命名空间分配标签。 这些命名空间由 keystore2 守护程序强制执行。密钥库始终都提供基于 UID/AID 的命名空间。密钥库 2.0 还会强制执行 sepolicy 定义的命名空间。如需详细了解此文件的格式和规范，请点击此处
根据日志生成命令
这些日志消息会指明在强制模式下哪些进程会失败以及失败原因。示例如下：
```
avc: denied  { connectto } for  pid=2671 comm="ping" path="/dev/socket/dnsproxyd"
scontext=u:r:shell:s0 tcontext=u:r:netd:s0 tclass=unix_stream_socket
```
该输出的解读如下：
- 上方的 { connectto } 表示执行的操作。根据它和末尾的 tclass (unix_stream_socket)，您可以大致了解是对什么对象执行什么操作。在此例中，是操作方正在试图连接到 UNIX 信息流套接字。
- scontext (u:r:shell:s0) 表示发起相应操作的环境，在此例中是 shell 中运行的某个程序。
- tcontext (u:r:netd:s0) 表示操作目标的环境，在此例中是归 netd 所有的某个 unix_stream_socket。
- 顶部的 comm="ping" 可帮助您了解拒绝事件发生时正在运行的程序。在此示例中，给出的信息非常清晰明了。
我们再看看另一个示例：
```
adb shell su root dmesg | grep 'avc: '
```
输出：
```
<5> type=1400 audit: avc:  denied  { read write } for  pid=177
comm="rmt_storage" name="mem" dev="tmpfs" ino=6004 scontext=u:r:rmt:s0
tcontext=u:object_r:kmem_device:s0 tclass=chr_file
```
以下是此拒绝事件的关键元素：
- 操作 - 试图进行的操作会使用括号突出显示：read write 或 setenforce。
- 操作方 - scontext（来源环境）条目表示操作方；在此例中为 rmt_storage 守护程序。
- 对象 - tcontext（目标环境）条目表示对哪个对象执行操作；在此例中为 kmem。
- 结果 - tclass（目标类别）条目表示操作对象的类型；在此例中为 chr_file（字符设备）。
实际解决问题操作步骤
抓取日志
```
dmesg|grep avc
或者
logcat|grep avc
```
生成政策命令
```
audit2allow -i aa.txt
```

日志
```
[    3.345194] audit: type=1400 audit(1896871430.404:5): avc:  denied  { open } for  pid=206 comm="recovery" path="/dev/__properties__/u:object_r:apexd_config_prop:s0" dev="tmpfs" ino=20 scontext=u:r:recovery:s0 tcontext=u:object_r:apexd_config_prop:s0 tclass=file permissive=1
[    3.345416] audit: type=1400 audit(1896871430.404:6): avc:  denied  { getattr } for  pid=206 comm="recovery" path="/dev/__properties__/u:object_r:apexd_config_prop:s0" dev="tmpfs" ino=20 scontext=u:r:recovery:s0 tcontext=u:object_r:apexd_config_prop:s0 tclass=file permissive=1
[    3.345627] audit: type=1400 audit(1896871430.404:7): avc:  denied  { map } for  pid=206 comm="recovery" path="/dev/__properties__/u:object_r:apexd_config_prop:s0" dev="tmpfs" ino=20 scontext=u:r:recovery:s0 tcontext=u:object_r:apexd_config_prop:s0 tclass=file permissive=1
[    3.345840] audit: type=1400 audit(1896871430.404:8): avc:  denied  { open } for  pid=206 comm="recovery" path="/dev/__properties__/u:object_r:apexd_prop:s0" dev="tmpfs" ino=21 scontext=u:r:recovery:s0 tcontext=u:object_r:apexd_prop:s0 tclass=file permissive=1
[    3.346041] audit: type=1400 audit(1896871430.404:9): avc:  denied  { getattr } for  pid=206 comm="recovery" path="/dev/__properties__/u:object_r:apexd_prop:s0" dev="tmpfs" ino=21 scontext=u:r:recovery:s0 tcontext=u:object_r:apexd_prop:s0 tclass=file permissive=1
[    3.346243] audit: type=1400 audit(1896871430.404:10): avc:  denied  { map } for  pid=206 comm="recovery" path="/dev/__properties__/u:object_r:apexd_prop:s0" dev="tmpfs" ino=21 scontext=u:r:recovery:s0 tcontext=u:object_r:apexd_prop:s0 tclass=file permissive=1
```
生成
#============= recovery ==============
```
allow recovery apexd_config_prop:file { getattr map };
allow recovery apexd_prop:file { getattr map open };
```
明确性
Android 并不会使用 SELinux 提供的所有功能。阅读外部文档时，请记住以下几点：
- AOSP 中的大部分政策都是使用内核政策语言定义的。在使用通用中间语言 (CIL) 时，会存在一些例外情况。
- 不使用 SELinux 用户。唯一定义的用户是 u
- 不使用 SELinux 角色和基于角色的访问权限控制 (RBAC)。定义并使用了两个默认角色：r（适用于主题）和 object_r（适用于对象）。
- 不使用 SELinux 敏感度。已始终设置好默认的 s0 敏感度。
- 不使用 SELinux 布尔值。一旦设备政策构建完成，该政策不再取决于设备状态。这简化了政策的审核和调试过程。
为新服务添加标签并解决拒绝事件
通过 init 启动的服务需要在各自的 SELinux 域中运行。以下示例会将服务“foo”放入它自己的 SELinux 网域中并为其授予权限。
该服务是在设备的 init.device.rc 文件中启动的，如下所示：
```
service foo /system/bin/foo
    class core
```
1.创建一个新网域“foo”
创建包含以下内容的文件 device/manufacturer/device-name/sepolicy/foo.te：
# foo service
```
type foo, domain;
type foo_exec, exec_type, file_type;

init_daemon_domain(foo)
```
这是 foo SELinux 网域的初始模板，您可以根据该可执行文件执行的具体操作为该模板添加规则。
2.为 /system/bin/foo 添加标签
将以下内容添加到 device/manufacturer/device-name/sepolicy/file_contexts：
/system/bin/foo   u:object_r:foo_exec:s0
这可确保为该可执行文件添加适当的标签，以便 SELinux 在适当的网域中运行相应服务。
3.构建并刷写启动映像和系统映像。
4.优化相应域的 SELinux 规则。

单编selinux
注意：开发中发现recovery 权限需要全编才能生效
```
souce/lunch
mmm system/sepolicy/
```
拷贝文件
```
adb push Z:\data1\T1EJ\buildsystem\android12\out\target\product\x9sp_ms_24MY\system\system_ext\etc\selinux /system/system_ext/etc/
adb push Z:\data1\T1EJ\buildsystem\android12\out\target\product\x9sp_ms_24MY\system\etc\selinux /system/etc/
adb push Z:\data1\T1EJ\buildsystem\android12\out\target\product\x9sp_ms_24MY\system\product\etc\selinux /system/product/etc/
adb push Z:\data1\T1EJ\buildsystem\android12\out\target\product\x9sp_ms_24MY\vendor\etc\selinux /vendor/etc/
```
# 参考
[官方文档](https://source.android.google.cn/docs/security/features/selinux/concepts?hl=zh-cn)
