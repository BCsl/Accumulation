# Tinker 热修复原理

## ClassLoader

在学习插件化的时候了解到 DexClassLoader 可以用来加载未安装应用的 dex 文件，用来执行非安装的程序代码，一般的热修复/插件化/MultiDex 技术是通过修改应用进程的 DexClassLoader 的加载 dex 的路径（对于插件化路径顺序没什么影响，对于热修复则需要置前，因为类是按照路径顺序遍历读取的）来动态对类进行修复/增加，对于插件化目的来说 dex 的增加并没什么影响，但对于热修复来说，就会有 `CLASS_ISPREVERIFIED` 标志的问题，[具体看这里](http://blog.csdn.net/lmj623565791/article/details/49883661)

## 生成 Patch

## Patch 的合并

```java
UpgradePatch.class

@Override
public boolean tryPatch(Context context, String tempPatchPath, PatchResult patchResult) {
    Tinker manager = Tinker.with(context);

    final File patchFile = new File(tempPatchPath);

    if (!manager.isTinkerEnabled() || !ShareTinkerInternals.isTinkerEnableWithSharedPreferences(context)) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:patch is disabled, just return");
        return false;
    }

    if (!SharePatchFileUtil.isLegalFile(patchFile)) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:patch file is not found, just return");
        return false;
    }
    //check the signature, we should create a new checker
    ShareSecurityCheck signatureCheck = new ShareSecurityCheck(context);

    int returnCode = ShareTinkerInternals.checkTinkerPackage(context, manager.getTinkerFlags(), patchFile, signatureCheck);
    if (returnCode != ShareConstants.ERROR_PACKAGE_CHECK_OK) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:onPatchPackageCheckFail");
        manager.getPatchReporter().onPatchPackageCheckFail(patchFile, returnCode);
        return false;
    }

    //it is a new patch, so we should not find a exist
    SharePatchInfo oldInfo = manager.getTinkerLoadResultIfPresent().patchInfo;
    String patchMd5 = SharePatchFileUtil.getMD5(patchFile);

    if (patchMd5 == null) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:patch md5 is null, just return");
        return false;
    }

    //use md5 as version
    patchResult.patchVersion = patchMd5;

    SharePatchInfo newInfo;

    //already have patch
    if (oldInfo != null) {
        if (oldInfo.oldVersion == null || oldInfo.newVersion == null) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:onPatchInfoCorrupted");
            manager.getPatchReporter().onPatchInfoCorrupted(patchFile, oldInfo.oldVersion, oldInfo.newVersion);
            return false;
        }

        if (!SharePatchFileUtil.checkIfMd5Valid(patchMd5)) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:onPatchVersionCheckFail md5 %s is valid", patchMd5);
            manager.getPatchReporter().onPatchVersionCheckFail(patchFile, oldInfo, patchMd5);
            return false;
        }
        newInfo = new SharePatchInfo(oldInfo.oldVersion, patchMd5, Build.FINGERPRINT);
    } else {
        newInfo = new SharePatchInfo("", patchMd5, Build.FINGERPRINT);
    }

    //check ok, we can real recover a new patch
    final String patchDirectory = manager.getPatchDirectory().getAbsolutePath();  // $PACKAGE_NAME/tinker/

    TinkerLog.i(TAG, "UpgradePatch tryPatch:patchMd5:%s", patchMd5);

    final String patchName = SharePatchFileUtil.getPatchVersionDirectory(patchMd5); //patch-MD5 的前八位

    final String patchVersionDirectory = patchDirectory + "/" + patchName;

    TinkerLog.i(TAG, "UpgradePatch tryPatch:patchVersionDirectory:%s", patchVersionDirectory);

    //copy file
    File destPatchFile = new File(patchVersionDirectory + "/" + SharePatchFileUtil.getPatchVersionFile(patchMd5));

    try {
        // check md5 first
        if (!patchMd5.equals(SharePatchFileUtil.getMD5(destPatchFile))) {
            //把 patch 文件拷贝到  $PACKAGE_NAME/tinker/patch-$(MD5 的前八位)/patch-$(MD5 的前八位).apk 这样的一个文件里
            SharePatchFileUtil.copyFileUsingStream(patchFile, destPatchFile);
        }
    } catch (IOException e) {
        //...
    }

    //we use destPatchFile instead of patchFile, because patchFile may be deleted during the patch process
    if (!DexDiffPatchInternal.tryRecoverDexFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile)) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, try patch dex failed");
        return false;
    }

    if (!BsDiffPatchInternal.tryRecoverLibraryFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile)) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, try patch library failed");
        return false;
    }

    if (!ResDiffPatchInternal.tryRecoverResourceFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile)) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, try patch resource failed");
        return false;
    }

    // check dex opt file at last, some phone such as VIVO/OPPO like to change dex2oat to interpreted
    // just warn
    if (!DexDiffPatchInternal.waitDexOptFile()) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, check dex opt file failed");
    }

    final File patchInfoFile = manager.getPatchInfoFile();

    if (!SharePatchInfo.rewritePatchInfoFileWithLock(patchInfoFile, newInfo, SharePatchFileUtil.getPatchInfoLockFile(patchDirectory))) {
        TinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, rewrite patch info failed");
        manager.getPatchReporter().onPatchInfoCorrupted(patchFile, newInfo.oldVersion, newInfo.newVersion);
        return false;
    }

    TinkerLog.w(TAG, "UpgradePatch tryPatch: done, it is ok");
    return true;
}
```
