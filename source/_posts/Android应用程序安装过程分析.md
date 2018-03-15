---
layout: pager
title: Android应用程序安装过程分析
date: 2018-03-15 21:56:46
tags: [Android,SystemService,PackageService]
description:  Android应用程序安装过程分析
---

Android应用程序安装过程分析
<!--more-->

```
应用程序管理服务PackageManagerService安装应用程序的过程，其实就是解析析应用程序配置文件AndroidManifest.xml的过程，并从里面得到得到应用程序的相关信息，例如得到应用程序的组件Activity、
Service、Broadcast Receiver和Content Provider等信息，有了这些信息后，通过ActivityManagerService这个服务，我们就可以在系统中正常地使用这些应用程序了。

应用程序管理服务PackageManagerService是系统启动的时候由SystemServer组件启动的，启后它就会执行应用程序安装的过程，因此，本文将从SystemServer启动PackageManagerService服务的过程
开始分析系统中的应用程序安装的过程。
```

本文从SystemServer开始分析，其实这也是PackageManagerService启动的过程
```java
/**
* The main entry point from zygote.
*/
public static void main(String[] args) {
    new SystemServer().run();
}	

SystemServer组件是由ZygoteInit.java启动的,具体的可以查看之前的分析文章 “Android系统启动流程(四)，进入SystemService”，最终会调用
在调用入口函数invokeStaticMain之前会将从ZygoteInit中传过来的参数进行处理，将com.android.server.SystemServer赋值给startClass，之后进入进程入口函数：也即是到了SystemService中的main函数

int main()
{
	....
	startBootstrapServices();
    startCoreServices();
    startOtherServices();
	...
}
在startBootstrapService中有
 private void startBootstrapServices() {
	....
	mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();
	.....
}
这个函数除了启动PackageManagerService服务之外，还启动了其它很多的服务，例如在前面学习Activity和Service的几篇文章中经常看到的ActivityManagerService服务，有兴趣的读者可以自己研究一下。
class PackageManagerService extends IPackageManager.Stub {  
    ......
	public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        ServiceManager.addService("package", m);
        return m;
    }
	......
}

这个函数创建了一个PackageManagerService服务实例，然后把这个服务添加到ServiceManager中去，ServiceManager是Android系统Binder进程间通信机制的守护进程，负责管理系统中的Binder对象
class PackageManagerService extends IPackageManager.Stub {  
	....
	public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
		....
		
		File dataDir = Environment.getDataDirectory();
        mAppInstallDir = new File(dataDir, "app");
        mAppLib32InstallDir = new File(dataDir, "app-lib");
        mEphemeralInstallDir = new File(dataDir, "app-ephemeral");
        mAsecInternalPath = new File(dataDir, "app-asec").getPath();
        mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
		File frameworkDir = new File(Environment.getRootDirectory(), "framework");
		
		// Collect vendor overlay packages.
        // (Do this before scanning any apps.)
        // For security and version matching reason, only consider
        // overlay packages if they reside in VENDOR_OVERLAY_DIR.
        File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
        scanDirTracedLI(vendorOverlayDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_TRUSTED_OVERLAY, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

        // Find base frameworks (resource packages without code).
        scanDirTracedLI(frameworkDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED,
                    scanFlags | SCAN_NO_DEX, 0);

        // Collected privileged system packages.
        final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
        scanDirTracedLI(privilegedAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

        // Collect ordinary system packages.
        final File systemAppDir = new File(Environment.getRootDirectory(), "app");
        scanDirTracedLI(systemAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

        // Collect all vendor packages.
        File vendorAppDir = new File("/vendor/app");
        try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
            scanDirTracedLI(vendorAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

        // Collect all OEM packages.
        final File oemAppDir = new File(Environment.getOemDirectory(), "app");
        scanDirTracedLI(oemAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
		
		....
	}
	....
}

Environment.getRootDirectory()  
public static File getRootDirectory() {
    return DIR_ANDROID_ROOT;
}
private static final File DIR_ANDROID_ROOT = getDirectory(ENV_ANDROID_ROOT, "/system");
private static final File DIR_ANDROID_DATA = getDirectory(ENV_ANDROID_DATA, "/data");

final File oemAppDir = new File(Environment.getOemDirectory(), "app");
public static File getOemDirectory() {
    return DIR_OEM_ROOT;
}
private static final File DIR_OEM_ROOT = getDirectory(ENV_OEM_ROOT, "/oem");

这里会调用scanDirLI函数来扫描移动设备上的下面这五个目录中的Apk文件：
/vendor/overlay

/system/framework

/system/priv-app

/system/app

/vendor/app
```

/system/app
![结果显示](/uploads/PackageManagerService/system-app.png)

/system/framework
![结果显示](/uploads/PackageManagerService/system-framewokr.png)

```java
继续执行 scanDirTracedLI函数
private void scanDirTracedLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanDir");
    try {
        scanDirLI(dir, parseFlags, scanFlags, currentTime);
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
}

 private void scanDirLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
    final File[] files = dir.listFiles();
	...
	for (File file : files) {
        final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
        if (!isPackage) {
             // Ignore entries which are not packages
            continue;
        }
        try {
            scanPackageTracedLI(file, parseFlags | PackageParser.PARSE_MUST_BE_APK,
                        scanFlags, currentTime, null);
        } catch (PackageManagerException e) {
                Slog.w(TAG, "Failed to parse " + file + ": " + e.getMessage());

                // Delete invalid userdata apps
            if ((parseFlags & PackageParser.PARSE_IS_SYSTEM) == 0 &&
                        e.error == PackageManager.INSTALL_FAILED_INVALID_APK) {
                    logCriticalInfo(Log.WARN, "Deleting invalid package at " + file);
                    removeCodePathLI(file);
            }
        }
    }
	...
}
public static final boolean isApkFile(File file) {
    return isApkPath(file.getName());
}

private static boolean isApkPath(String path) {
    return path.endsWith(".apk");
}
对于目录中的每一个文件，如果是以后Apk作为后缀名，那么就调用scanPackageLI函数来对它进行解析和安装。

private PackageParser.Package scanPackageTracedLI(File scanFile, final int parseFlags,
    int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanPackage");
    try {
        return scanPackageLI(scanFile, parseFlags, scanFlags, currentTime, user);
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
}

private PackageParser.Package scanPackageLI(File scanFile, int parseFlags, int scanFlags,
            long currentTime, UserHandle user) throws PackageManagerException {
    if (DEBUG_INSTALL) Slog.d(TAG, "Parsing: " + scanFile);
    PackageParser pp = new PackageParser();
    pp.setSeparateProcesses(mSeparateProcesses);
    pp.setOnlyCoreApps(mOnlyCore);
    pp.setDisplayMetrics(mMetrics);

    if ((scanFlags & SCAN_TRUSTED_OVERLAY) != 0) {
            parseFlags |= PackageParser.PARSE_TRUSTED_OVERLAY;
    }

    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
    final PackageParser.Package pkg;
    try {
        pkg = pp.parsePackage(scanFile, parseFlags);
    } catch (PackageParserException e) {
        throw PackageManagerException.from(e);
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
    return scanPackageLI(pkg, scanFile, parseFlags, scanFlags, currentTime, user);
}

这个函数首先会为这个Apk文件创建一个PackageParser实例，接着调用这个实例的parsePackage函数来对这个Apk文件进行解析。
这个函数最后还会调用另外一个版本的scanPackageLI函数把来解析后得到的应用程序信息保存在PackageManagerService中。

先来看parsePackage()函数
public Package parsePackage(File packageFile, int flags) throws PackageParserException {
    if (packageFile.isDirectory()) {
        return parseClusterPackage(packageFile, flags);
    } else {
        return parseMonolithicPackage(packageFile, flags);
    }
}

public Package parseMonolithicPackage(File apkFile, int flags) throws PackageParserException {
    final PackageLite lite = parseMonolithicPackageLite(apkFile, flags);
    if (mOnlyCoreApps) {
        if (!lite.coreApp) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                        "Not a coreApp: " + apkFile);
        }
    }

    final AssetManager assets = new AssetManager();
    try {
        final Package pkg = parseBaseApk(apkFile, assets, flags);
        pkg.setCodePath(apkFile.getAbsolutePath());
        pkg.setUse32bitAbi(lite.use32bitAbi);
        return pkg;
    } finally {
        IoUtils.closeQuietly(assets);
    }
}

private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
            throws PackageParserException {
        final String apkPath = apkFile.getAbsolutePath();
		
	....
	Resources res = null;
    XmlResourceParser parser = null;
    try {
        res = new Resources(assets, mMetrics, null);
        assets.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,Build.VERSION.RESOURCES_SDK_INT);
        parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

        final String[] outError = new String[1];
        final Package pkg = parseBaseApk(res, parser, flags, outError);
        if (pkg == null) {
                throw new PackageParserException(mParseError,
                        apkPath + " (at " + parser.getPositionDescription() + "): " + outError[0]);
            }

        pkg.setVolumeUuid(volumeUuid);
        pkg.setApplicationVolumeUuid(volumeUuid);
        pkg.setBaseCodePath(apkPath);
        pkg.setSignatures(null);
        return pkg;
	} catch (PackageParserException e) {
		
	}
	.....
}

private static final String ANDROID_MANIFEST_FILENAME = "AndroidManifest.xml";
通过AndroidManaifest.xml构造一个解析器,然后执行 parseBaseApk

 private Package parseBaseApk(Resources res, XmlResourceParser parser, int flags,
            String[] outError) throws XmlPullParserException, IOException {
    final String splitName;
    final String pkgName;

     try {
            Pair<String, String> packageSplit = parsePackageSplitNames(parser, parser);
            pkgName = packageSplit.first;
            splitName = packageSplit.second;

            if (!TextUtils.isEmpty(splitName)) {
                outError[0] = "Expected base APK, but found split " + splitName;
                mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME;
                return null;
            }
        } catch (PackageParserException e) {
            mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME;
            return null;
        }

        final Package pkg = new Package(pkgName);

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifest);

        pkg.mVersionCode = pkg.applicationInfo.versionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
        pkg.baseRevisionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_revisionCode, 0);
        pkg.mVersionName = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_versionName, 0);
        if (pkg.mVersionName != null) {
            pkg.mVersionName = pkg.mVersionName.intern();
        }

        pkg.coreApp = parser.getAttributeBooleanValue(null, "coreApp", false);

        sa.recycle();

        return parseBaseApkCommon(pkg, null, res, parser, flags, outError);
}

得到VersionCode等，然后执行parseBaseApkCommon（）函数
private Package parseBaseApkCommon(Package pkg, Set<String> acceptedTags, Resources res,
            XmlResourceParser parser, int flags, String[] outError) throws XmlPullParserException,
            IOException {
	....
	int outerDepth = parser.getDepth();
    while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();

            if (acceptedTags != null && !acceptedTags.contains(tagName)) {
                Slog.w(TAG, "Skipping unsupported element under <manifest>: "
                        + tagName + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }

            if (tagName.equals(TAG_APPLICATION)) {
                if (foundApp) {
                    if (RIGID_PARSER) {
                        outError[0] = "<manifest> has more than one <application>";
                        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                        return null;
                    } else {
                        Slog.w(TAG, "<manifest> has more than one <application>");
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                }

                foundApp = true;
                if (!parseBaseApplication(pkg, res, parser, flags, outError)) {
                    return null;
         }
		...
		 } else if (tagName.equals(TAG_KEY_SETS)) {
                if (!parseKeySets(pkg, res, parser, outError)) {
                    return null;
                }
            } else if (tagName.equals(TAG_PERMISSION_GROUP)) {
                if (parsePermissionGroup(pkg, flags, res, parser, outError) == null) {
                    return null;
                }
            } else if (tagName.equals(TAG_PERMISSION)) {
                if (parsePermission(pkg, res, parser, outError) == null) {
                    return null;
                }
            } else if (tagName.equals(TAG_PERMISSION_TREE)) {
                if (parsePermissionTree(pkg, res, parser, outError) == null) {
                    return null;
                }
            } else if (tagName.equals(TAG_USES_PERMISSION)) {
                if (!parseUsesPermission(pkg, res, parser)) {
                    return null;
                }
            } else if (tagName.equals(TAG_USES_PERMISSION_SDK_M)
                    || tagName.equals(TAG_USES_PERMISSION_SDK_23)) {
                if (!parseUsesPermission(pkg, res, parser)) {
                    return null;
                }
		....
	....				
}

这里就是对AndroidManifest.xml文件中的各个标签进行解析了，这里我们只简单看一下application标签的解析，这是通过调用parseBaseApplication函数来进行的。

 private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        final ApplicationInfo ai = owner.applicationInfo;
        final String pkgName = owner.applicationInfo.packageName;
	
	....
	final int innerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);

            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.receivers.add(a);

            } else if (tagName.equals("service")) {
                Service s = parseService(owner, res, parser, flags, outError);
                if (s == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.services.add(s);

            } else if (tagName.equals("provider")) {
                Provider p = parseProvider(owner, res, parser, flags, outError);
                if (p == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.providers.add(p);

            } else if (tagName.equals("activity-alias")) {
                Activity a = parseActivityAlias(owner, res, parser, flags, outError);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);
	.....	
}


这里就是对AndroidManifest.xml文件中的application标签进行解析了，我们常用到的标签就有activity、service、receiver和provider
这里解析完成后，回到之前，调用另一个版本的scanPackageLI函数把来解析后得到的应用程序信息保存下来。
 private PackageParser.Package scanPackageLI(PackageParser.Package pkg, File scanFile,
            final int policyFlags, int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {
			
	....
	// Scan the parent
    PackageParser.Package scannedPkg = scanPackageInternalLI(pkg, scanFile, policyFlags,
                scanFlags, currentTime, user);

        // Scan the children
    final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
    for (int i = 0; i < childCount; i++) {
        PackageParser.Package childPackage = pkg.childPackages.get(i);
        scanPackageInternalLI(childPackage, scanFile, policyFlags, scanFlags,
        currentTime, user);
    }


    if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
        return scanPackageLI(pkg, scanFile, policyFlags, scanFlags, currentTime, user);
    }
	
	....
}

 private PackageParser.Package scanPackageInternalLI(PackageParser.Package pkg, File scanFile,
            int policyFlags, int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {
        PackageSetting ps = null;
        PackageSetting updatedPkg;
	....
	// Note that we invoke the following method only if we are about to unpack an application
    PackageParser.Package scannedPkg = scanPackageLI(pkg, policyFlags, scanFlags | SCAN_UPDATE_SIGNATURE, currentTime, user);
	.....
}

private PackageParser.Package scanPackageLI(PackageParser.Package pkg, final int policyFlags,
            int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
        boolean success = false;
        try {
            final PackageParser.Package res = scanPackageDirtyLI(pkg, policyFlags, scanFlags,
                    currentTime, user);
            success = true;
            return res;
        } finally {
            if (!success && (scanFlags & SCAN_DELETE_DATA_ON_FAILURES) != 0) {
                // DELETE_DATA_ON_FAILURES is only used by frozen paths
                destroyAppDataLIF(pkg, UserHandle.USER_ALL,
                        StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                destroyAppProfilesLIF(pkg, UserHandle.USER_ALL);
            }
        }
}

 private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
            final int policyFlags, final int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {
	....
			N = pkg.services.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Service s = pkg.services.get(i);
                s.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        s.info.processName, pkg.applicationInfo.uid);
                mServices.addService(s);
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(s.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Services: " + r);
            }

            N = pkg.receivers.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.receivers.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mReceivers.addActivity(a, "receiver");
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(a.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Receivers: " + r);
            }

            N = pkg.activities.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.activities.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mActivities.addActivity(a, "activity");
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(a.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Activities: " + r);
            }

	....			
}

/ All available activities, for your resolving pleasure.
final ActivityIntentResolver mActivities = new ActivityIntentResolver();

// All available receivers, for your resolving pleasure.
final ActivityIntentResolver mReceivers = new ActivityIntentResolver();

// All available services, for your resolving pleasure.
final ServiceIntentResolver mServices = new ServiceIntentResolver();

// All available providers, for your resolving pleasure.
final ProviderIntentResolver mProviders = new ProviderIntentResolver();


这个函数主要就是把前面解析应用程序得到的package、provider、service、receiver和activity等信息保存在PackageManagerService服务中了。
```
