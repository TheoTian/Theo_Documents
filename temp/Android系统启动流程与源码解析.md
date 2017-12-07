###Android开机启动步骤:
1. 引导程序启动init进程
2. init进程通过解析init.zygotexx.rc文件启动app_process

	```
	service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
	```
3. app_process启动Zygote进程

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

1. 通知ActivityManagerService启动新进程

	```
	 Process.ProcessStartResult startResult = Process.start(entryPoint,
	                    app.processName, uid, uid, gids, debugFlags, mountExternal,
	                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
	                    app.info.dataDir, entryPointArgs);
	```
	
2. 通过Zygote启动新的进程

	```
	    return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
	```