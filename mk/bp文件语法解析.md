# Android.bp

## 基础知识讲解

[官方文档](https://source.android.google.cn/setup/build?hl=zh_cn#variables)

## 实际开发中经验归纳

``` 
    srcs: [
        "app/src/main/java/**/*.java",
    ], //当前路径默认android.bp所在路径
    
     resource_dirs: [
        "app/src/main/res",
    ],  //当前路径默认android.bp所在路径
    
    manifest: "app/src/main/AndroidManifest.xml", //当前路径默认android.bp所在路径
    
    rtti: true, //开启rtti，经验证在cflags中加-frtti是无效的
    
    include_dirs:[
        "frameworks/base/core/jni",
    ], //当前路径默认源码根目录
    
``` 
