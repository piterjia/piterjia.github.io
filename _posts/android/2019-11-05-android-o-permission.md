---
layout: post
title:  Android O 运行时权限策略变化
categories: Android
description: Android O的运行时权限策略变化
keywords: Android, Android 8.0, 权限
---

以下是Android官方文档

> 在 Android 8.0 之前，如果应用在运行时请求权限并且被授予该权限，系统会错误地将属于同一权限组并且在清单中注册的其他权限也一起授予应用。
> 对于针对 Android 8.0 的应用，此行为已被纠正。系统只会授予应用明确请求的权限。然而，一旦用户为应用授予某个权限，则所有后续对该权限组中权限的请求都将被自动批准。
> 例如，假设某个应用在其清单中列出 READ_EXTERNAL_STORAGE 和 WRITE_EXTERNAL_STORAGE。应用请求 READ_EXTERNAL_STORAGE，并且用户授予了该权限。如果该应用针对的是 API 级别 24 或更低级别，系统还会同时授予 WRITE_EXTERNAL_STORAGE，因为该权限也属于同一 STORAGE 权限组并且也在清单中注册过。如果该应用针对的是 Android 8.0，则系统此时仅会授予 READ_EXTERNAL_STORAGE；不过，如果该应用后来又请求 WRITE_EXTERNAL_STORAGE，则系统会立即授予该权限，而不会提示用户。



基于此，在权限处理时需要特别注意两点:

1. 需要的权限要明确申请，不要依赖同组之内的其他权限的申请，比如需要读写权限，WRITE_EXTERNAL_STORAGE和 READ_EXTERNAL_STORAGE全部申请，即便在属于同一组。
2. 只要有需要运行时权限的地方就要申请，不依赖之前流程的权限申请情况，因为之前很可能用户没有同意此权限，调用便会发生异常。


在我们的项目中，采用自定义组件 **PermissionChecker** 来申请运行时权限，需要新增权限时用法如下：

```
new PermissionChecker().checkPermissions(permissions, new PermissionResult() {
    @Override
 public void onResult(boolean agree) {
        
 if (agree) {
                //do something.//涉及系统权限，务必在用户同意权限后再处理事务，继 agree 为true。
 } 
       } 
});
```

在agree为true，代表用户同意授权此权限，在这种情况下，再进行下一步的业务处理。
