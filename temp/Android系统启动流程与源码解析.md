###Android开机启动步骤:
1. 引导程序启动init进程
2. init进程通过解析init.zygotexx.rc文件启动app_process进程
```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
```
3. app_process进程启动Zygote进程
```
runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
```
4. zygote进程fork systemserver进程
```
 pid = Zygote.forkSystemServer(parsedArgs.uid, parsedArgs.gid,parsedArgs.gids,parsedArgs.debugFlags,null,parsedArgs.permittedCapabilities,parsedArgs.effectiveCapabilities);
```
5. systemserver进程开启子线程启动各个service如pms和ams等,同时会开启ActivityThread.

	```
	 // Initialize the system context.
    createSystemContext();
    // Create the system service manager.
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    ...
	 startBootstrapServices();
	 startCoreServices();
	 startOtherServices();
	```


###Android启动用户进程流程
