---
title: dylib动态库加载过程分析
date: 2017-01-14 09:42:58
tags: macOS软件安全
post_asset_folder: true
top: 100
---

Windows系统的动态库是DLL文件，Linux系统是so文件，macOS系统的动态库则使用dylib文件作为动态库。
dylib本质上是一个Mach-O格式的文件，它与普通的Mach-O执行文件几乎使用一样的结构，只是在文件类型上一个是`MH_DYLIB`，一个是`MH_EXECUTE`。
在系统的/usr/lib目录下，存放了大量供系统与应用程序调用的动态库文件，使用`file`命令查看系统动态库libobjc.dylib的信息，输出如下：
```
$ file /usr/lib/libobjc.dylib
/usr/lib/libobjc.dylib: Mach-O universal binary with 3 architectures
/usr/lib/libobjc.dylib (for architecture i386):	Mach-O dynamically linked shared library i386
/usr/lib/libobjc.dylib (for architecture x86_64):	Mach-O 64-bit dynamically linked shared library x86_64
/usr/lib/libobjc.dylib (for architecture x86_64h):	Mach-O 64-bit dynamically linked shared library x86_64
```
从上面的输出信息可以看出，libobjc.dylib是一个通用的二进制文件，包含了三种cpu架构的Mach-O。另外，
可以使用Mach-O格式文件管理工具`otool`查看dylib的信息，如查看动态库的依赖库信息如下：
```
$ otool -L /usr/lib/libobjc.dylib
/usr/lib/libobjc.dylib:
	/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
	/usr/lib/libauto.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/libc++abi.dylib (compatibility version 1.0.0, current version 125.0.0)
	/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 120.1.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1225.0.0)
```

## 0x1 构建动态库
`XCode`环境提供了创建动态库的工程模板，创建动态库的方法比较简单，在`XCode`中选择File->New->Project，在打开的工程模选择对话框中，选择标签macOS->Framework & Library，在右侧选择Library，点击Next按钮，在新页面中输入项目名称mylib，Type选择Dynamic，单击Next按钮选择项目保存的路径后，工程就创建好了。接着修改工程文件内容：
```
//mylib.h
#import <Foundation/Foundation.h>

@interface mylib : NSObject
-(void) hello;
@end

//mylib.m
#import "mylib.h"

@implementation mylib
-(void) hello {
    NSLog(@"hello world");
}
@end
```
保存后。觇击菜单Product->Build，或者按键般的COMMAND+B键就编译成功了。命令执行完后，就会生成mylib.dylib文件。
XCode创建的项目是xcodeproj文件，可以使用XCode提供的工具xcodebuild在命令行下编译，在命令行下切换到工程文件所在的目录后，执行`xcodebuild`会有如下输出：
```
$ xcodebuild
=== BUILD TARGET mylib OF PROJECT mylib WITH THE DEFAULT CONFIGURATION (Release) ===

Check dependencies

Write auxiliary files
write-file /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/mylib-own-target-headers.hmap
write-file /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/mylib-all-non-framework-target-headers.hmap
......
/bin/mkdir -p /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/Objects-normal/x86_64
write-file /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/Objects-normal/x86_64/mylib.LinkFileList

CompileC build/mylib.build/Release/mylib.build/Objects-normal/x86_64/mylib.o mylib/mylib.m normal x86_64 objective-c com.apple.compilers.llvm.clang.1_0.compiler
    cd /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib
    export LANG=en_US.US-ASCII
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -arch x86_64 -fmessage-length=94 -fdiagnostics-show-note-include-stack -fmacro-backtrace-limit=0 -fcolor-diagnostics -std=gnu99 -fobjc-arc -fmodules -gmodules -fmodules-prune-interval=86400 -fmodules-prune-after=345600 -fbuild-session-file=/var/folders/rd/mts0362j0n92rq0z1cnmdb580000gn/C/org.llvm.clang/ModuleCache/Session.modulevalidation -fmodules-validate-once-per-build-session -Wnon-modular-include-in-framework-module -Werror=non-modular-include-in-framework-module -Wno-trigraphs -fpascal-strings -Os -fno-common -Wno-missing-field-initializers -Wno-missing-prototypes -Werror=return-type -Wunreachable-code -Wno-implicit-atomic-properties -Werror=deprecated-objc-isa-usage -Werror=objc-root-class -Wno-arc-repeated-use-of-weak -Wduplicate-method-match -Wno-missing-braces -Wparentheses -Wswitch -Wunused-function -Wno-unused-label -Wno-unused-parameter -Wunused-variable -Wunused-value -Wempty-body -Wconditional-uninitialized -Wno-unknown-pragmas -Wno-shadow -Wno-four-char-constants -Wno-conversion -Wconstant-conversion -Wint-conversion -Wbool-conversion -Wenum-conversion -Wshorten-64-to-32 -Wpointer-sign -Wno-newline-eof -Wno-selector -Wno-strict-selector-match -Wundeclared-selector -Wno-deprecated-implementations -DNS_BLOCK_ASSERTIONS=1 -DOBJC_OLD_DISPATCH_PROTOTYPES=0 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.11.sdk -fasm-blocks -fstrict-aliasing -Wprotocol -Wdeprecated-declarations -mmacosx-version-min=10.11 -g -Wno-sign-conversion -iquote /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/mylib-generated-files.hmap -I/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/mylib-own-target-headers.hmap -I/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/mylib-all-target-headers.hmap -iquote /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/mylib-project-headers.hmap -I/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release/include -I/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/DerivedSources/x86_64 -I/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/DerivedSources -F/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release -MMD -MT dependencies -MF /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/Objects-normal/x86_64/mylib.d --serialize-diagnostics /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/Objects-normal/x86_64/mylib.dia -c /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/mylib/mylib.m -o /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/Objects-normal/x86_64/mylib.o

Ld build/Release/libmylib.dylib normal x86_64
    cd /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib
    export MACOSX_DEPLOYMENT_TARGET=10.11
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -arch x86_64 -dynamiclib -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.11.sdk -L/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release -F/Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release -filelist /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/Objects-normal/x86_64/mylib.LinkFileList -install_name /usr/local/lib/libmylib.dylib -mmacosx-version-min=10.11 -fobjc-arc -fobjc-link-runtime -single_module -compatibility_version 1 -current_version 1 -Xlinker -dependency_info -Xlinker /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/mylib.build/Release/mylib.build/Objects-normal/x86_64/mylib_dependency_info.dat -o /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release/libmylib.dylib

GenerateDSYMFile build/Release/libmylib.dylib.dSYM build/Release/libmylib.dylib
    cd /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/dsymutil /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release/libmylib.dylib -o /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release/libmylib.dylib.dSYM

CodeSign build/Release/libmylib.dylib
    cd /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib
    export CODESIGN_ALLOCATE=/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/codesign_allocate

Signing Identity:     "-"

    /usr/bin/codesign --force --sign - --timestamp=none /Users/macbook/Documents/macbook/macbook/code/chapter4/mylib/build/Release/libmylib.dylib

** BUILD SUCCEEDED **
```
从上面的日志中可以看出，整个编译过程分为：
检查依赖（Check dependencies）、生成辅助文件（Write auxiliary files）、编译（CompileC）、链接（Ld）、生成调试符号（GenerateDSYMFile）、代码签名（CodeSign）等几步。
编译代码时，使用的编译器是`clang`，这是苹果公司开发的用来替代`gcc`的现代化编译器，该编译器目前也广泛用于安卓、Linux平台上的软件开发工作；链接时使用`clang`前端传入参数给链接器`ld`，链接完成后dylib动态库就编译成功了；生成调试符号这一步主要用于生成符号的调试信息，供调试器使用；最后一步是代码签名，在没有指定签名证书的情况下，XCode默认使用的adhoc签名。

编译好的动态库可以被其它程序通过头文件声明隐式的调用，也可以像Linux系统那样，使用系统函数`dlopen()`、`dlsym()`手动进行调用。

## 0x2 dyld
动态库不能直接运行，而是需要通过系统的动态链接加载器进行加载到内存后执行，动态链接加载器在系统中以一个用户态的可执行文件形式存在，一般应用程序会在Mach-O文件部分指定一个`LC_LOAD_DYLINKER`的加载命令，此加载命令指定了dyld的路径，通常它的默认值是“/usr/lib/dyld”。系统内核在加载Mach-O文件时，会使用该路径指定的程序作为动态库的加载器来加载dylib。

dyld加载时，为了优化程序启动，启用了共享缓存（shared cache）技术。共享缓存会在进程启动时被dyld映射到内存中，之后，当任何Mach-O映像加载时，dyld首先会检查该Mach-O映像与所需的动态库是否在共享缓存中，如果存在，则直接将它在共享内存中的内存地址映射到进程的内存地址空间。在程序依赖的系统动态库很多的情况下，这种做法对程序启动性能是有明显提升的。

`update_dyld_shared_cache`程序确保了dyld的共享缓存是最新的，它会扫描/var/db/dyld/shared_region_roots/目录下paths路径文件，这些paths文件包含了需要加入到共享缓存的Mach-O文件路径列表，`update_dyld_shared_cache()`会挨个将这些Mach-O文件及其依赖的dylib都加共享缓存中去。

共享缓存是以文件形式存放在/var/db/dyld/目录下的，生成共享缓存的`update_dyld_shared_cache`程序位于是/usr/bin/目录下，该工具会为每种系统加构生成一个缓存文件与对应的内存地址map表，如下所示：
```
ls -l /var/db/dyld/
total 1741296
-rw-r--r--   1 root  wheel  333085108 Apr 22 15:02 dyld_shared_cache_i386
-rw-r--r--   1 root  wheel      65378 Apr 22 15:02 dyld_shared_cache_i386.map
-rw-r--r--   1 root  wheel  558259294 Apr 25 16:18 dyld_shared_cache_x86_64h
-rw-r--r--   1 root  wheel     129633 Apr 25 16:18 dyld_shared_cache_x86_64h.map
drwxr-xr-x  10 root  wheel        340 Apr  7 09:19 shared_region_roots
```
生成的共享缓存可以使用工具`dyld_shared_cache_util`查看它的信息，该工具位于dyld源码中的 launch-cache\dyld_shared_cache_util.cpp 文件，需要自己手动编译。另外，也可以使用dyld提供的两个函数`dyld_shared_cache_extract_dylibs()`与`dyld_shared_cache_extract_dylibs_progress()`来自己解开cache文件，代码位于dyld源码的launch-cache\dsc_extractor.cpp文件中。

`update_dyld_shared_cache`通常它只在系统的安装器安装软件与系统更新时调用，当然，可以手动运行“sudo update_dyld_shared_cache”来更新共享缓存。新的共享缓存会在系统下次启动后自动更新。

## 0x3 动态库的加载过程分析
dyld是苹果操作系统一个重要组成部分，而且令人兴奋的是，它是开源的，任何人可以通过苹果官网下载它的源码来阅读理解它的运作方式（下载地址：http://opensource.apple.com/tarballs/dyld），了解系统加载动态库的细节。

系统内核在加载动态库前，会加载dyld，然后调用去执行`__dyld_start()`，该函数会执行`dyldbootstrap::start()`，后者会执行`_main()`函数，dyld的加载动态库的代码就是从`_main()`开始执行的。下面以dyld源码的360.18版本为蓝本进行分析：
```
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide,
		int argc, const char* argv[], const char* envp[], const char* apple[],
		uintptr_t* startGlue)
{
    //第一步，设置运行环境，处理环境变量
	uintptr_t result = 0;
	sMainExecutableMachHeader = mainExecutableMH;

    ......

	CRSetCrashLogMessage("dyld: launch started");

    ......

	setContext(mainExecutableMH, argc, argv, envp, apple);

	// Pickup the pointer to the exec path.
	sExecPath = _simple_getenv(apple, "executable_path");

	// <rdar://problem/13868260> Remove interim apple[0] transition code from dyld
	if (!sExecPath) sExecPath = apple[0];

	......

	sExecShortName = ::strrchr(sExecPath, '/');
	if ( sExecShortName != NULL )
		++sExecShortName;
	else
		sExecShortName = sExecPath;
    sProcessIsRestricted = processRestricted(mainExecutableMH, &ignoreEnvironmentVariables, &sProcessRequiresLibraryValidation);
    if ( sProcessIsRestricted ) {
#if SUPPORT_LC_DYLD_ENVIRONMENT
		checkLoadCommandEnvironmentVariables();
#endif 	
		pruneEnvironmentVariables(envp, &apple);
		setContext(mainExecutableMH, argc, argv, envp, apple);
	}
	else {
		if ( !ignoreEnvironmentVariables )
			checkEnvironmentVariables(envp);
		defaultUninitializedFallbackPaths(envp);
	}
	if ( sEnv.DYLD_PRINT_OPTS )
		printOptions(argv);
	if ( sEnv.DYLD_PRINT_ENV )
		printEnvironmentVariables(envp);
	getHostInfo(mainExecutableMH, mainExecutableSlide);

    ......


    //第二步，初始化主程序
	try {
		// add dyld itself to UUID list
		addDyldImageToUUIDList();
		CRSetCrashLogMessage(sLoadingCrashMessage);
		// instantiate ImageLoader for main executable
		sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);
		gLinkContext.mainExecutable = sMainExecutable;
		gLinkContext.processIsRestricted = sProcessIsRestricted;
		gLinkContext.processRequiresLibraryValidation = sProcessRequiresLibraryValidation;
		gLinkContext.mainExecutableCodeSigned = hasCodeSignatureLoadCommand(mainExecutableMH);

        ......

		//第三步，加载共享缓存
		checkSharedRegionDisable();
	#if DYLD_SHARED_CACHE_SUPPORT
		if ( gLinkContext.sharedRegionMode != ImageLoader::kDontUseSharedRegion )
			mapSharedCache();
	#endif

		// Now that shared cache is loaded, setup an versioned dylib overrides
	#if SUPPORT_VERSIONED_PATHS
		checkVersionedPaths();
	#endif

		//第四步，加载插入的动态库
		if	( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
			for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib)
				loadInsertedDylib(*lib);
		}
		sInsertedDylibCount = sAllImages.size()-1;

		//第五步，链接主程序
		gLinkContext.linkingMainExecutable = true;
		link(sMainExecutable, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
		sMainExecutable->setNeverUnloadRecursive();
		if ( sMainExecutable->forceFlat() ) {
			gLinkContext.bindFlat = true;
			gLinkContext.prebindUsage = ImageLoader::kUseNoPrebinding;
		}

		//第六步，链接插入的动态库
		if ( sInsertedDylibCount > 0 ) {
			for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
				ImageLoader* image = sAllImages[i+1];
				link(image, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
				image->setNeverUnloadRecursive();
			}
			// only INSERTED libraries can interpose
			for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
				ImageLoader* image = sAllImages[i+1];
				image->registerInterposing();
			}
		}

		// <rdar://problem/19315404> dyld should support interposition even without DYLD_INSERT_LIBRARIES
		for (int i=sInsertedDylibCount+1; i < sAllImages.size(); ++i) {
			ImageLoader* image = sAllImages[i];
			if ( image->inSharedCache() )
				continue;
			image->registerInterposing();
		}

		// apply interposing to initial set of images
		for(int i=0; i < sImageRoots.size(); ++i) {
			sImageRoots[i]->applyInterposing(gLinkContext);
		}

        //第七步，执行弱符号绑定
		gLinkContext.linkingMainExecutable = false;
		// <rdar://problem/12186933> do weak binding only after all inserted images linked
		sMainExecutable->weakBind(gLinkContext);

        //第八步，执行初始化方法
		CRSetCrashLogMessage("dyld: launch, running initializers");
	#if SUPPORT_OLD_CRT_INITIALIZATION
		// Old way is to run initializers via a callback from crt1.o
		if ( ! gRunInitializersOldWay )
			initializeMainExecutable();
	#else
		// run all initializers
		initializeMainExecutable();
	#endif

        //第九步，查找入口点并返回
		result = (uintptr_t)sMainExecutable->getThreadPC();
		if ( result != 0 ) {
			// main executable uses LC_MAIN, needs to return to glue in libdyld.dylib
			if ( (gLibSystemHelpers != NULL) && (gLibSystemHelpers->version >= 9) )
				*startGlue = (uintptr_t)gLibSystemHelpers->startGlueToCallExit;
			else
				halt("libdyld.dylib support not present for LC_MAIN");
		}
		else {
			// main executable uses LC_UNIXTHREAD, dyld needs to let "start" in program set up for main()
			result = (uintptr_t)sMainExecutable->getMain();
			*startGlue = 0;
		}
	}
	catch(const char* message) {
		syncAllImages();
		halt(message);
	}
	catch(...) {
		dyld::log("dyld: launch failed\n");
	}

	CRSetCrashLogMessage(NULL);

	return result;
}
```
整个方法的代码比较长，将它按功能分成九个步骤进行讲解：

#### 第一步，设置运行环境，处理环境变量

代码在开始时候，将传入的变量`mainExecutableMH`赋值给了`sMainExecutableMachHeader`，这是一个`macho_header`类型的变量，其结构体内容就是本章前面介绍的`mach_header`结构体，表示的是当前主程序的Mach-O头部信息，有了头部信息，加载器就可以从头开始，遍历整个Mach-O文件的信息。
接着执行了`setContext()`，此方法设置了全局一个链接上下文，包括一些回调函数、参数与标志设置信息，代码片断如下：
```
static void setContext(const macho_header* mainExecutableMH, int argc, const char* argv[], const char* envp[], const char* apple[])
{
	gLinkContext.loadLibrary			= &libraryLocator;
	gLinkContext.terminationRecorder	= &terminationRecorder;
	gLinkContext.flatExportFinder		= &flatFindExportedSymbol;
	gLinkContext.coalescedExportFinder	= &findCoalescedExportedSymbol;
	gLinkContext.getCoalescedImages		= &getCoalescedImages;

    ......

	gLinkContext.bindingOptions			= ImageLoader::kBindingNone;
	gLinkContext.argc					= argc;
	gLinkContext.argv					= argv;
	gLinkContext.envp					= envp;
	gLinkContext.apple					= apple;
	gLinkContext.progname				= (argv[0] != NULL) ? basename(argv[0]) : "";
	gLinkContext.programVars.mh			= mainExecutableMH;
	gLinkContext.programVars.NXArgcPtr	= &gLinkContext.argc;
	gLinkContext.programVars.NXArgvPtr	= &gLinkContext.argv;
	gLinkContext.programVars.environPtr	= &gLinkContext.envp;
	gLinkContext.programVars.__prognamePtr=&gLinkContext.progname;
	gLinkContext.mainExecutable			= NULL;
	gLinkContext.imageSuffix			= NULL;
	gLinkContext.dynamicInterposeArray	= NULL;
	gLinkContext.dynamicInterposeCount	= 0;
	gLinkContext.prebindUsage			= ImageLoader::kUseAllPrebinding;
#if TARGET_IPHONE_SIMULATOR
	gLinkContext.sharedRegionMode		= ImageLoader::kDontUseSharedRegion;
#else
	gLinkContext.sharedRegionMode		= ImageLoader::kUseSharedRegion;
#endif
}
```
设置的回调函数都是dyld本模块实现的，如`loadLibrary`方法就是本模块的`libraryLocator()`方法，负责加载动态库。
在设置完这些信息后，执行`processRestricted()`方法判断进程是否受限。代码如下：
```
static bool processRestricted(const macho_header* mainExecutableMH, bool* ignoreEnvVars, bool* processRequiresLibraryValidation)
{
#if TARGET_IPHONE_SIMULATOR
	gLinkContext.codeSigningEnforced = true;
#else
    // ask kernel if code signature of program makes it restricted
    uint32_t flags;
	if ( csops(0, CS_OPS_STATUS, &flags, sizeof(flags)) != -1 ) {
		if (flags & CS_REQUIRE_LV)
			*processRequiresLibraryValidation = true;

  #if __MAC_OS_X_VERSION_MIN_REQUIRED
		if ( flags & CS_ENFORCEMENT ) {
			gLinkContext.codeSigningEnforced = true;
		}
		if ( ((flags & CS_RESTRICT) == CS_RESTRICT) && (csr_check(CSR_ALLOW_TASK_FOR_PID) != 0) ) {
			sRestrictedReason = restrictedByEntitlements;
			return true;
		}
  #else
		if ((flags & CS_ENFORCEMENT) && !(flags & CS_GET_TASK_ALLOW)) {
			*ignoreEnvVars = true;
		}
		gLinkContext.codeSigningEnforced = true;
  #endif
	}
#endif

	// all processes with setuid or setgid bit set are restricted
    if ( issetugid() ) {
		sRestrictedReason = restrictedBySetGUid;
		return true;
	}

	// <rdar://problem/13158444&13245742> Respect __RESTRICT,__restrict section for root processes
	if ( hasRestrictedSegment(mainExecutableMH) ) {
		// existence of __RESTRICT/__restrict section make process restricted
		sRestrictedReason = restrictedBySegment;
		return true;
	}
    return false;
}
```
进程受限会是以下三种可能：
`restrictedByEntitlements`：在macOS系统上，在需要验证代码签名（`Gatekeeper`开启）的情况下，且`csr_check(CSR_ALLOW_TASK_FOR_PID)`返回为真（表示Rootless开启了`TASK_FOR_PID`标志）时，进程才不会受限，在macOS版本10.12系统上，默认`Gatekeeper`是开启的，并且`Rootless`是关闭了`CSR_ALLOW_TASK_FOR_PID`标志位的，这意味着，默认情况下，系统上运行的进程是受限的。
`restrictedBySetGUid`：当进程的setuid与setgid位被设置时，进程会被设置成受限。这样做是出于安全的考虑，受限后的进程无法访问`DYLD_`开头的环境变量，一种典型的系统攻击就是针对这种情况而发生的，在macOS版本10.10系统上，一个由`DYLD_PRINT_TO_FILE`环境变量引发的系统本地提权漏洞，就是通过向`DYLD_PRINT_TO_FILE`环境变量传入拥有SUID权限的受限文件，而系统没做安全检测，而这些文件是直接有向系统创建与写入文件权限的。关于漏洞的具体细节可以参看：https://www.sektioneins.de/en/blog/15-07-07-dyld_print_to_file_lpe.html
`restrictedBySegment`：段名受限。当Mach-O包含一个`__RESTRICT/__restrict`段时，进程会被设置成受限。

在进程受限后，执行了以下三个方法：
`checkLoadCommandEnvironmentVariables()`:遍历Mach-O中所有的`LC_DYLD_ENVIRONMENT`加载命令，然后调用`processDyldEnvironmentVariable()`对不同的环境变量做相应的处理。
`pruneEnvironmentVariables()`:删除进程的`LD_LIBRARY_PATH`与所有以DYLD_开头的环境变量，这样以后创建的子进程就不包含这些环境变量了。
`setContext()`:重新设置链接上下文。这一步执行的主要目的是由于环境变量发生变化了，需要更新进程的`envp`与`apple`参数。


#### 第二步，初始化主程序
这一步主要执行了`instantiateFromLoadedImage()`。它的代码如下：
```
static ImageLoader* instantiateFromLoadedImage(const macho_header* mh, uintptr_t slide, const char* path)
{
	// try mach-o loader
	if ( isCompatibleMachO((const uint8_t*)mh, path) ) {
		ImageLoader* image = ImageLoaderMachO::instantiateMainExecutable(mh, slide, path, gLinkContext);
		addImage(image);
		return image;
	}

	throw "main executable not a known format";
}
```
`isCompatibleMachO()`主要检查Mach-O的头部的`cputype`与`cpusubtype`来判断程序与当前的系统是否兼容。如果兼容接下来就调用`instantiateMainExecutable()`实例化主程序，代码如下：
```
ImageLoader* ImageLoaderMachO::instantiateMainExecutable(const macho_header* mh, uintptr_t slide, const char* path, const LinkContext& context)
{
	//dyld::log("ImageLoader=%ld, ImageLoaderMachO=%ld, ImageLoaderMachOClassic=%ld, ImageLoaderMachOCompressed=%ld\n",
	//	sizeof(ImageLoader), sizeof(ImageLoaderMachO), sizeof(ImageLoaderMachOClassic), sizeof(ImageLoaderMachOCompressed));
	bool compressed;
	unsigned int segCount;
	unsigned int libCount;
	const linkedit_data_command* codeSigCmd;
	const encryption_info_command* encryptCmd;
	sniffLoadCommands(mh, path, false, &compressed, &segCount, &libCount, context, &codeSigCmd, &encryptCmd);
	// instantiate concrete class based on content of load commands
	if ( compressed )
		return ImageLoaderMachOCompressed::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
	else
#if SUPPORT_CLASSIC_MACHO
		return ImageLoaderMachOClassic::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
#else
		throw "missing LC_DYLD_INFO load command";
#endif
}
```
`sniffLoadCommands()`主要获取了加载命令中的如下信息：
`compressed`：判断Mach-O的`Compressed`还是`Classic`类型。判断的依据是Mach-O是否包含`LC_DYLD_INFO`或`LC_DYLD_INFO_ONLY`加载命令。这2个加载命令记录了Mach-O的动态库加载信息，使用结构体`dyld_info_command`表示：
```
struct dyld_info_command {
   uint32_t   cmd;		/* LC_DYLD_INFO or LC_DYLD_INFO_ONLY */
   uint32_t   cmdsize;		/* sizeof(struct dyld_info_command) */
   uint32_t   rebase_off;	/* file offset to rebase info  */
   uint32_t   rebase_size;	/* size of rebase info   */
   uint32_t   bind_off;	/* file offset to binding info   */
   uint32_t   bind_size;	/* size of binding info  */
   uint32_t   weak_bind_off;	/* file offset to weak binding info   */
   uint32_t   weak_bind_size;  /* size of weak binding info  */
   uint32_t   lazy_bind_off;	/* file offset to lazy binding info */
   uint32_t   lazy_bind_size;  /* size of lazy binding infs */
   uint32_t   export_off;	/* file offset to lazy binding info */
   uint32_t   export_size;	/* size of lazy binding infs */
};
```
`rebase_off`与大小`rebase_size`存储了rebase（重设基址）相关信息，当Mach-O加载到内存中的地址不是指定的首选地址时，就需要对当前的映像数据进行rebase（重设基址）。
`bind_off`与`bind_size`存储了进程的符号绑定信息，当进程启动时必须绑定这些符号，典型的有`dyld_stub_binder`，该符号被dyld用来做迟绑定加载符号，一般动态库都包含该符号。
`weak_bind_off`与`weak_bind_size`存储了进程的弱绑定符号信息。弱符号主要用于面向对旬语言中的符号重载，典型的有c++中使用new创建对象，默认情况下会绑定ibstdc++.dylib，如果检测到某个映像使用弱符号引用重载了`new`符号，dyld则会重新绑定该符号并调用重载的版本。
`lazy_bind_off`与`lazy_bind_size`存储了进程的延迟绑定符号信息。有些符号在进程启动时不需要马上解析，它们会在第一次调用时被解析，这类符号叫延迟绑定符号（Lazy Symbol）。
`export_off`与`export_size`存储了进程的导出符号绑定信息。导出符号可以被外部的Mach-O访问，通常动态库会导出一个或多个符号供外部使用，而可执行程序由导出`_main`与`_mh_execute_header`符号供dyld使用。

`segCount`：段的数量。`sniffLoadCommands()`通过遍历所有的`LC_SEGMENT_COMMAND`加载命令来获取段的数量。

`libCount`：需要加载的动态库的数量。Mach-O中包含的每一条`LC_LOAD_DYLIB`、`LC_LOAD_WEAK_DYLIB`、`LC_REEXPORT_DYLIB`、`LC_LOAD_UPWARD_DYLIB`加载命令，都表示需要加载一个动态库。

`codeSigCmd`：通过解析`LC_CODE_SIGNATURE`来获取代码签名的加载命令。

`encryptCmd`：通过`LC_ENCRYPTION_INFO`与`LC_ENCRYPTION_INFO_64`来获取段加密信息。

获取`compressed`后，根据Mach-O是否`compressed`来分别调用`ImageLoaderMachOCompressed::instantiateMainExecutable()`与`ImageLoaderMachOClassic::instantiateMainExecutable()`。`ImageLoaderMachOCompressed::instantiateMainExecutable()`代码如下：
```
// create image for main executable
ImageLoaderMachOCompressed* ImageLoaderMachOCompressed::instantiateMainExecutable(const macho_header* mh, uintptr_t slide, const char* path,
																		unsigned int segCount, unsigned int libCount, const LinkContext& context)
{
	ImageLoaderMachOCompressed* image = ImageLoaderMachOCompressed::instantiateStart(mh, path, segCount, libCount);

	// set slide for PIE programs
	image->setSlide(slide);

	// for PIE record end of program, to know where to start loading dylibs
	if ( slide != 0 )
		fgNextPIEDylibAddress = (uintptr_t)image->getEnd();

	image->disableCoverageCheck();
	image->instantiateFinish(context);
	image->setMapped(context);

	if ( context.verboseMapping ) {
		dyld::log("dyld: Main executable mapped %s\n", path);
		for(unsigned int i=0, e=image->segmentCount(); i < e; ++i) {
			const char* name = image->segName(i);
			if ( (strcmp(name, "__PAGEZERO") == 0) || (strcmp(name, "__UNIXSTACK") == 0)  )
				dyld::log("%18s at 0x%08lX->0x%08lX\n", name, image->segPreferredLoadAddress(i), image->segPreferredLoadAddress(i)+image->segSize(i));
			else
				dyld::log("%18s at 0x%08lX->0x%08lX\n", name, image->segActualLoadAddress(i), image->segActualEndAddress(i));
		}
	}

	return image;
}
```
`ImageLoaderMachOCompressed::instantiateStart()`使用主程序Mach-O信息构造了一个`ImageLoaderMachOCompressed`对象。`disableCoverageCheck()`禁用覆盖率检查。`instantiateFinish()`调用`parseLoadCmds()`解析其它所有的加载命令，后者会填充完`ImageLoaderMachOCompressed`的一些保护成员信息，最后调用`setDyldInfo()`设置动态库链接信息，然后调用`setSymbolTableInfo()`设置符号表信息。

`instantiateFromLoadedImage()`调用完了`ImageLoaderMachO::instantiateMainExecutable()`后，接着调用`addImage()`，代码如下：
```
static void addImage(ImageLoader* image)
{
	// add to master list
    allImagesLock();
        sAllImages.push_back(image);
    allImagesUnlock();

	// update mapped ranges
	uintptr_t lastSegStart = 0;
	uintptr_t lastSegEnd = 0;
	for(unsigned int i=0, e=image->segmentCount(); i < e; ++i) {
		if ( image->segUnaccessible(i) )
			continue;
		uintptr_t start = image->segActualLoadAddress(i);
		uintptr_t end = image->segActualEndAddress(i);
		if ( start == lastSegEnd ) {
			// two segments are contiguous, just record combined segments
			lastSegEnd = end;
		}
		else {
			// non-contiguous segments, record last (if any)
			if ( lastSegEnd != 0 )
				addMappedRange(image, lastSegStart, lastSegEnd);
			lastSegStart = start;
			lastSegEnd = end;
		}		
	}
	if ( lastSegEnd != 0 )
		addMappedRange(image, lastSegStart, lastSegEnd);


	if ( sEnv.DYLD_PRINT_LIBRARIES || (sEnv.DYLD_PRINT_LIBRARIES_POST_LAUNCH && (sMainExecutable!=NULL) && sMainExecutable->isLinked()) ) {
		dyld::log("dyld: loaded: %s\n", image->getPath());
	}

}
```
这段代码将实例化好的主程序添加到全局主列表`sAllImages`中，最后调用`addMappedRange()`申请内存，更新主程序映像映射的内存区。做完这些工作，第二步初始化主程序就算完成了。


#### 第三步，加载共享缓存
这一步主要执行`mapSharedCache()`来映射共享缓存。该函数先通过`_shared_region_check_np()`来检查缓存是否已经映射到了共享区域了，如果已经映射了，就更新缓存的`slide`与`UUID`，然后返回。反之，判断系统是否处于安全启动模式（safe-boot mode）下，如果是就删除缓存文件并返回，正常启动的情况下，接下来调用`openSharedCacheFile()`打开缓存文件，该函数在`sSharedCacheDir`路径下，打开与系统当前cpu架构匹配的缓存文件，也就是/var/db/dyld/dyld_shared_cache_x86_64h，接着读取缓存文件的前8192字节，解析缓存头`dyld_cache_header`的信息，将解析好的缓存信息存入`mappings`变量，最后调用`_shared_region_map_and_slide_np()`完成真正的映射工作。部分代码片断如下：
```
static void mapSharedCache()
{
	uint64_t cacheBaseAddress = 0;
	if ( _shared_region_check_np(&cacheBaseAddress) == 0 ) {
		sSharedCache = (dyld_cache_header*)cacheBaseAddress;
#if __x86_64__
		const char* magic = (sHaswell ? ARCH_CACHE_MAGIC_H : ARCH_CACHE_MAGIC);
#else
		const char* magic = ARCH_CACHE_MAGIC;
#endif
		if ( strcmp(sSharedCache->magic, magic) != 0 ) {
			sSharedCache = NULL;
			if ( gLinkContext.verboseMapping ) {
				dyld::log("dyld: existing shared cached in memory is not compatible\n");
				return;
			}
		}
		const dyld_cache_header* header = sSharedCache;
		......
		// if cache has a uuid, copy it
		if ( header->mappingOffset >= 0x68 ) {
			memcpy(dyld::gProcessInfo->sharedCacheUUID, header->uuid, 16);
		}
		......
	}
	else {
#if __i386__ || __x86_64__
		uint32_t	safeBootValue = 0;
		size_t		safeBootValueSize = sizeof(safeBootValue);
		if ( (sysctlbyname("kern.safeboot", &safeBootValue, &safeBootValueSize, NULL, 0) == 0) && (safeBootValue != 0) ) {
			struct stat dyldCacheStatInfo;
			if ( my_stat(MACOSX_DYLD_SHARED_CACHE_DIR DYLD_SHARED_CACHE_BASE_NAME ARCH_NAME, &dyldCacheStatInfo) == 0 ) {
				struct timeval bootTimeValue;
				size_t bootTimeValueSize = sizeof(bootTimeValue);
				if ( (sysctlbyname("kern.boottime", &bootTimeValue, &bootTimeValueSize, NULL, 0) == 0) && (bootTimeValue.tv_sec != 0) ) {
					if ( dyldCacheStatInfo.st_mtime < bootTimeValue.tv_sec ) {
						::unlink(MACOSX_DYLD_SHARED_CACHE_DIR DYLD_SHARED_CACHE_BASE_NAME ARCH_NAME);
						gLinkContext.sharedRegionMode = ImageLoader::kDontUseSharedRegion;
						return;
					}
				}
			}
		}
#endif
		// map in shared cache to shared region
		int fd = openSharedCacheFile();
		if ( fd != -1 ) {
			uint8_t firstPages[8192];
			if ( ::read(fd, firstPages, 8192) == 8192 ) {
				dyld_cache_header* header = (dyld_cache_header*)firstPages;
		#if __x86_64__
				const char* magic = (sHaswell ? ARCH_CACHE_MAGIC_H : ARCH_CACHE_MAGIC);
		#else
				const char* magic = ARCH_CACHE_MAGIC;
		#endif
				if ( strcmp(header->magic, magic) == 0 ) {
					const dyld_cache_mapping_info* const fileMappingsStart = (dyld_cache_mapping_info*)&firstPages[header->mappingOffset];
					const dyld_cache_mapping_info* const fileMappingsEnd = &fileMappingsStart[header->mappingCount];
					shared_file_mapping_np	mappings[header->mappingCount+1]; // add room for code-sig

					......

						if (_shared_region_map_and_slide_np(fd, mappingCount, mappings, codeSignatureMappingIndex, cacheSlide, slideInfo, slideInfoSize) == 0) {
							sSharedCache = (dyld_cache_header*)mappings[0].sfm_address;
							sSharedCacheSlide = cacheSlide;
							dyld::gProcessInfo->sharedCacheSlide = cacheSlide;
							......
						}
						else {
#if __IPHONE_OS_VERSION_MIN_REQUIRED
							throw "dyld shared cache could not be mapped";
#endif
							if ( gLinkContext.verboseMapping )
								dyld::log("dyld: shared cached file could not be mapped\n");
						}
					}
				}
				else {
					if ( gLinkContext.verboseMapping )
						dyld::log("dyld: shared cached file is invalid\n");
				}
			}
			else {
				if ( gLinkContext.verboseMapping )
					dyld::log("dyld: shared cached file cannot be read\n");
			}
			close(fd);
		}
		else {
			if ( gLinkContext.verboseMapping )
				dyld::log("dyld: shared cached file cannot be opened\n");
		}
	}
	......
}
```
共享缓存加载完毕后，接着进行动态库的版本化重载，这主要通过函数checkVersionedPaths()完成。该函数读取`DYLD_VERSIONED_LIBRARY_PATH`与`DYLD_VERSIONED_FRAMEWORK_PATH`环境变量，将指定版本的库比当前加载的库的版本做比较，如果当前的库版本更高的话，就使用新版本的库来替换掉旧版本的。


#### 第四步，加载插入的动态库
这一步循环遍历`DYLD_INSERT_LIBRARIES`环境变量中指定的动态库列表，并调用`loadInsertedDylib()`将其加载。该函数调用`load()`完成加载工作。`load()`会调用`loadPhase0()`尝试从文件加载，`loadPhase0()`会向下调用下一层phase来查找动态库的路径，直到`loadPhase6()`，查找的顺序为`DYLD_ROOT_PATH`->`LD_LIBRARY_PATH`->`DYLD_FRAMEWORK_PATH`->原始路径->`DYLD_FALLBACK_LIBRARY_PATH`，找到后调用`ImageLoaderMachO::instantiateFromFile()`来实例化一个`ImageLoader`，之后调用`checkandAddImage()`验证映像并将其加入到全局映像列表中。如果`loadPhase0()`返回为空，表示在路径中没有找到动态库，就尝试从共享缓存中查找，找到就调用`ImageLoaderMachO::instantiateFromCache()`从缓存中加载，否则就抛出没找到映像的异常。部分代码片断如下：
```
ImageLoader* load(const char* path, const LoadContext& context)
{
	......
	if ( context.useSearchPaths && ( gLinkContext.imageSuffix != NULL) ) {
		if ( realpath(path, realPath) != NULL )
			path = realPath;
	}

	ImageLoader* image = loadPhase0(path, orgPath, context, NULL);
	if ( image != NULL ) {
		CRSetCrashLogMessage2(NULL);
		return image;
	}

	......
	image = loadPhase0(path, orgPath, context, &exceptions);
#if __IPHONE_OS_VERSION_MIN_REQUIRED && DYLD_SHARED_CACHE_SUPPORT && !TARGET_IPHONE_SIMULATOR
	// <rdar://problem/16704628> support symlinks on disk to a path in dyld shared cache
	if ( (image == NULL) && cacheablePath(path) && !context.dontLoad ) {
		......
		if ( (myerr == ENOENT) || (myerr == 0) )
		{
			const macho_header* mhInCache;
			const char*			pathInCache;
			long				slideInCache;
			if ( findInSharedCacheImage(resolvedPath, false, NULL, &mhInCache, &pathInCache, &slideInCache) ) {
				struct stat stat_buf;
				bzero(&stat_buf, sizeof(stat_buf));
				try {
					image = ImageLoaderMachO::instantiateFromCache(mhInCache, pathInCache, slideInCache, stat_buf, gLinkContext);
					image = checkandAddImage(image, context);
				}
				catch (...) {
					image = NULL;
				}
			}
		}
	}
#endif
    ......
	else {
		const char* msgStart = "no suitable image found.  Did find:";
		......
		throw (const char*)fullMsg;
	}
}
```


#### 第五步，链接主程序
这一步执行`link()`完成主程序的链接操作。该函数调用了`ImageLoader`自身的`link()`函数，主要的目的是将实例化的主程序的动态数据进行修正，达到让进程可用的目的，典型的就是主程序中的符号表修正操作，它的代码片断如下：
```
void ImageLoader::link(const LinkContext& context, bool forceLazysBound, bool preflightOnly, bool neverUnload, const RPathChain& loaderRPaths)
{
	......
	this->recursiveLoadLibraries(context, preflightOnly, loaderRPaths);

	......

	context.clearAllDepths();
	this->recursiveUpdateDepth(context.imageCount());

 	this->recursiveRebase(context);

	......

 	this->recursiveBind(context, forceLazysBound, neverUnload);

	if ( !context.linkingMainExecutable )
		this->weakBind(context);	//现在是链接主程序，这里现在不会执行

	......

	std::vector<DOFInfo> dofs;
	this->recursiveGetDOFSections(context, dofs);
	context.registerDOFs(dofs);

	......

	if ( !context.linkingMainExecutable && (fgInterposingTuples.size() != 0) ) {
		this->recursiveApplyInterposing(context);	//现在是链接主程序，这里现在不会执行
	}
	......
}
```
`recursiveLoadLibraries()`采用递归的方式来加载程序依赖的动态库，加载的方法是调用`context`的`loadLibrary`指针方法，该方法在前面看到过，是`setContext()`设置的`libraryLocator()`，该函数只是调用了`load()`来完成加载，`load()`加载动态库的过程在上一步已经分析过了。

接着调用`recursiveUpdateDepth()`对映像及其依赖库按列表方式进行排序。`recursiveRebase()`则对映像完成递归rebase操作，该函数只是调用了虚函数`doRebase()`，`doRebase()`被`ImageLoaderMachO`重载，实际只是将代码段设置成可写后调用了`rebase()`，在`ImageLoaderMachOCompressed`中，该函数读取映像动态链接信息的`rebase_off`与`rebase_size`来确定需要rebase的数据偏移与大小，然后挨个修正它们的地址信息。

`recursiveBind()`完成递归绑定符号表的操作。此处的符号表针对的是非延迟加载的符号表，它的核心是调用了`doBind()`，在`ImageLoaderMachOCompressed`中，该函数读取映像动态链接信息的`bind_off`与`bind_size`来确定需要绑定的数据偏移与大小，然后挨个对它们进行绑定，绑定操作具体使用`bindAt()`函数，它主要通过调用`resolve()`解析完符号表后，调用`bindLocation()`完成最终的绑定操作，需要绑定的符号信息有三种：
`BIND_TYPE_POINTER`：需要绑定的是一个指针。直接将计算好的新值屿值即可。
`BIND_TYPE_TEXT_ABSOLUTE32`：一个32位的值。取计算的值的低32位赋值过去。
`BIND_TYPE_TEXT_PCREL32`：重定位符号。需要使用新值减掉需要修正的地址值来计算出重定位值。

`recursiveGetDOFSections`()与`registerDOFs()`主要注册程序的DOF节区，供`dtrace`使用。


#### 第六步，链接插入的动态库
链接插入的动态库与链接主程序一样，都是使用的`link()`,插入的动态库列表是前面调用`addImage()`保存到`sAllImages`中的，之后，循环获取每一个动态库的`ImageLoader`，调用`link()`对其进行链接，注意：`sAllImages`中保存的第一项是主程序的映像。接下来调用每个映像的`registerInterposing()`方法来注册动态库插入与调用`applyInterposing()`应用插入操作。`registerInterposing()`查找`__DATA`段的`__interpose`节区，找到需要应用插入操作（也可以叫作符号地址替换）的数据，然后做一些检查后，将要替换的符号与被替换的符号信息存入`fgInterposingTuples`列表中，供以后具体符号替换时查询。`applyInterposing()`调用了虚方法`doInterpose()`来做符号替换操作，在`ImageLoaderMachOCompressed`中实际是调用了`eachBind()`与`eachLazyBind()`分别对常规的符号与延迟加载的符号进行应用插入操作，具体使用的是`interposeAt()`，该方法调用`interposedAddress()`在`fgInterposingTuples`中查找要替换的符号地址，找到后然后进行最终的符号地址替换。


#### 第七步，执行弱符号绑定
`weakBind()`函数执行弱符号绑定。首先通过调用`context`的`getCoalescedImages()`将`sAllImages`中所有含有弱符号的映像合并成一个列表，合并完后调用`initializeCoalIterator()`对映像进行排序，排序完成后调用`incrementCoalIterator()`收集需要进行绑定的弱符号，后者是一个虚函数，在`ImageLoaderMachOCompressed`中，该函数读取映像动态链接信息的`weak_bind_off`与`weak_bind_size`来确定弱符号的数据偏移与大小，然后挨个计算它们的地址信息。之后调用`getAddressCoalIterator()`，按照映像的加载顺序在导出表中查找符号的地址，找到后调用`updateUsesCoalIterator()`执行最终的绑定操作，执行绑定的是`bindLocation()`，前面有讲过，此处不再赘述。


#### 第八步，执行初始化方法
执行初始化的方法是`initializeMainExecutable()`。该函数主要执行`runInitializers()`，后者调用了`ImageLoader`的`runInitializers()`方法，最终迭代执行了`ImageLoaderMachO`的`doInitialization()`方法，后者主要调用`doImageInit()`与`doModInitFunctions()`执行映像与模块中设置为init的函数与静态初始化方法，代码如下：
```
bool ImageLoaderMachO::doInitialization(const LinkContext& context)
{
	CRSetCrashLogMessage2(this->getPath());

	doImageInit(context);
	doModInitFunctions(context);

	CRSetCrashLogMessage2(NULL);

	return (fHasDashInit || fHasInitializers);
}
```


#### 第九步，查找入口点并返回
这一步调用主程序映像的`getThreadPC()`函数来查找主程序的`LC_MAIN`加载命令获取程序的入口点，没找到就调用`getMain()`到`LC_UNIXTHREAD`加载命令中去找，找到后就跳到入口点指定的地址并返回了。

到这里，dyld整个加载动态库的过程就算完成了。

另外再讨论下延迟符号加载的技术细节。在所有拥有延迟加载符号的Mach-O文件里，它的符号表中一定有一个`dyld_stub_helper`符号，它是延迟符号加载的关键！延迟绑定符号的修正工作就是由它完成的。绑定符号信息可以使用`XCode`提供的命令行工具`dyldinfo`来查看，执行以下命令可以查看`python`的绑定信息：
```
xcrun dyldinfo -bind /usr/bin/python
for arch i386:
bind information:
segment section          address        type    addend dylib            symbol
__DATA  __cfstring       0x000040F0    pointer      0 CoreFoundation   ___CFConstantStringClassReference
__DATA  __cfstring       0x00004100    pointer      0 CoreFoundation   ___CFConstantStringClassReference
__DATA  __nl_symbol_ptr  0x00004010    pointer      0 CoreFoundation   _kCFAllocatorNull
__DATA  __nl_symbol_ptr  0x00004008    pointer      0 libSystem        ___stack_chk_guard
__DATA  __nl_symbol_ptr  0x0000400C    pointer      0 libSystem        _environ
__DATA  __nl_symbol_ptr  0x00004000    pointer      0 libSystem        dyld_stub_binder
bind information:
segment section          address        type    addend dylib            symbol
__DATA  __cfstring       0x1000031D8    pointer      0 CoreFoundation   ___CFConstantStringClassReference
__DATA  __cfstring       0x1000031F8    pointer      0 CoreFoundation   ___CFConstantStringClassReference
__DATA  __got            0x100003010    pointer      0 CoreFoundation   _kCFAllocatorNull
__DATA  __got            0x100003000    pointer      0 libSystem        ___stack_chk_guard
__DATA  __got            0x100003008    pointer      0 libSystem        _environ
__DATA  __nl_symbol_ptr  0x100003018    pointer      0 libSystem        dyld_stub_binder
```

所有的延迟绑定符号都存储在`_TEXT`段的`stubs`节区（桩节区），编译器在生成代码时创建的符号调用就生成在此节区中，该节区被称为“桩”节区，桩只是一小段临时使用的指令，在`stubs`中只是一条`jmp`跳转指令，跳转的地址位于`__DATA`段`__la_symbol_ptr`节区中，指向的是一段代码，类似于如下的语句：
```
push xxx
jmp yyy
```
其中xxx是符号在动态链接信息中延迟绑定符号数据的偏移值，yyy则是跳转到`_TEXT`段的`stub_helper`节区头部，此处的代码通常为：
```
lea        r11, qword [ds:zzz]
push       r11
jmp        qword [ds:imp___nl_symbol_ptr_dyld_stub_binder]
```
`jmp`跳转的地址是`__DATA`段中`__nl_symbol_ptr`节区，指向的是符号`dyld_stub_binder()`，该函数由dyld导出，实现位于dyld源码的“dyld_stub_binder.s”文件中，它调用`dyld::fastBindLazySymbol()`来绑定延迟加载的符号，后者是一个虚函数，实际调用`ImageLoaderMachOCompressed`的`doBindFastLazySymbol()`，后者调用`bindAt()`解析并返回正确的符号地址，`dyld_stub_binder()`在最后跳转到符号地址去执行。这一步完成后，`__DATA`段`__la_symbol_ptr`节区中存储的符号地址就是修正后的地址，下一次调用该符号时，就直接跳转到真正的符号地址去执行，而不用`dyld_stub_binder()`来重新解析该符号了，
