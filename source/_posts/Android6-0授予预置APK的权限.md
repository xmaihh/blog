---

title: Android6.0授予预置APK的权限
date: 2018-07-28 17:02:40
categories: Android Framework
tags: [Android]
toc: false
description: If you're prepared to adapt and learn, you can transform.

---

对我们系统中存在的应用进行默认权限设置,达到默认开启应用权限无需申请权限弹框的目的
方法1
修改`\frameworks\base\services\core\java\com\android\server\pm\PackageManagerService.java`，但CTS会有问题

```
 private void grantPermissionsLPw(PackageParser.Package pkg, boolean replace,
            String packageOfInterest) {
        // IMPORTANT: There are two types of permissions: install and runtime.
        // Install time permissions are granted when the app is installed to
        // all device users and users added in the future. Runtime permissions
        // are granted at runtime explicitly to specific users. Normal and signature
        // protected permissions are install time permissions. Dangerous permissions
        // are install permissions if the app's target SDK is Lollipop MR1 or older,
        // otherwise they are runtime permissions. This function does not manage
        // runtime permissions except for the case an app targeting Lollipop MR1
        // being upgraded to target a newer SDK, in which case dangerous permissions
        // are transformed from install time to runtime ones.

        final PackageSetting ps = (PackageSetting) pkg.mExtras;
        if (ps == null) {
            return;
        }

        PermissionsState permissionsState = ps.getPermissionsState();
        PermissionsState origPermissions = permissionsState;

        final int[] currentUserIds = UserManagerService.getInstance().getUserIds();

        boolean runtimePermissionsRevoked = false;
        int[] changedRuntimePermissionUserIds = EMPTY_INT_ARRAY;

        boolean changedInstallPermission = false;

        if (replace) {
            ps.installPermissionsFixed = false;
            if (!ps.isSharedUser()) {
                origPermissions = new PermissionsState(permissionsState);
                permissionsState.reset();
            } else {
                // We need to know only about runtime permission changes since the
                // calling code always writes the install permissions state but
                // the runtime ones are written only if changed. The only cases of
                // changed runtime permissions here are promotion of an install to
                // runtime and revocation of a runtime from a shared user.
                changedRuntimePermissionUserIds = revokeUnusedSharedUserPermissionsLPw(
                        ps.sharedUser, UserManagerService.getInstance().getUserIds());
                if (!ArrayUtils.isEmpty(changedRuntimePermissionUserIds)) {
                    runtimePermissionsRevoked = true;
                }
            }
        }

        permissionsState.setGlobalGids(mGlobalGids);

        final int N = pkg.requestedPermissions.size();
        for (int i=0; i<N; i++) {
            final String name = pkg.requestedPermissions.get(i);
            final BasePermission bp = mSettings.mPermissions.get(name);

            if (DEBUG_INSTALL) {
                Log.i(TAG, "Package " + pkg.packageName + " checking " + name + ": " + bp);
            }

            if (bp == null || bp.packageSetting == null) {
                if (packageOfInterest == null || packageOfInterest.equals(pkg.packageName)) {
                    Slog.w(TAG, "Unknown permission " + name
                            + " in package " + pkg.packageName);
                }
                continue;
            }

            final String perm = bp.name;
            boolean allowedSig = false;
            int grant = GRANT_DENIED;

            // Keep track of app op permissions.
            if ((bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_APPOP) != 0) {
                ArraySet<String> pkgs = mAppOpPermissionPackages.get(bp.name);
                if (pkgs == null) {
                    pkgs = new ArraySet<>();
                    mAppOpPermissionPackages.put(bp.name, pkgs);
                }
                pkgs.add(pkg.packageName);
            }

            final int level = bp.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE;
            switch (level) {
                case PermissionInfo.PROTECTION_NORMAL: {
                    // For all apps normal permissions are install time ones.
                    grant = GRANT_INSTALL;
                } break;

                case PermissionInfo.PROTECTION_DANGEROUS: {
                    if (pkg.applicationInfo.targetSdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1){
                        // For legacy apps dangerous permissions are install time ones.
                        grant = GRANT_INSTALL_LEGACY;
                    } else if (origPermissions.hasInstallPermission(bp.name)) {
                        // For legacy apps that became modern, install becomes runtime.
                        grant = GRANT_UPGRADE;
                    } else if (mPromoteSystemApps
                            && isSystemApp(ps)
                            && mExistingSystemPackages.contains(ps.name)) {
                        // For legacy system apps, install becomes runtime.
                        // We cannot check hasInstallPermission() for system apps since those
                        // permissions were granted implicitly and not persisted pre-M.
                        grant = GRANT_UPGRADE;
                    } else {
                        // For modern apps keep runtime permissions unchanged.
                        grant = GRANT_RUNTIME;
                    }
                } break;

                case PermissionInfo.PROTECTION_SIGNATURE: {
                    // For all apps signature permissions are install time ones.
                    allowedSig = grantSignaturePermission(perm, pkg, bp, origPermissions);
                    if (allowedSig) {
                        grant = GRANT_INSTALL;
                    }
                } break;
            }

            if (DEBUG_INSTALL) {
                Log.i(TAG, "Package " + pkg.packageName + " granting " + perm);
            }

            if (grant != GRANT_DENIED) {
                if (!isSystemApp(ps) && ps.installPermissionsFixed) {
                    // If this is an existing, non-system package, then
                    // we can't add any new permissions to it.
                    if (!allowedSig && !origPermissions.hasInstallPermission(perm)) {
                        // Except...  if this is a permission that was added
                        // to the platform (note: need to only do this when
                        // updating the platform).
                        if (!isNewPlatformPermissionForPackage(perm, pkg)) {
                            grant = GRANT_DENIED;
                        }
                    }
                }

                switch (grant) {
                    case GRANT_INSTALL: {
                        // Revoke this as runtime permission to handle the case of
                        // a runtime permission being downgraded to an install one.
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                            if (origPermissions.getRuntimePermissionState(
                                    bp.name, userId) != null) {
                                // Revoke the runtime permission and clear the flags.
                                origPermissions.revokeRuntimePermission(bp, userId);
                                origPermissions.updatePermissionFlags(bp, userId,
                                      PackageManager.MASK_PERMISSION_FLAGS, 0);
                                // If we revoked a permission permission, we have to write.
                                changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                        changedRuntimePermissionUserIds, userId);
                            }
                        }
                        // Grant an install permission.
                        if (permissionsState.grantInstallPermission(bp) !=
                                PermissionsState.PERMISSION_OPERATION_FAILURE) {
                            changedInstallPermission = true;
                        }
                    } break;

                    case GRANT_INSTALL_LEGACY: {
                        // Grant an install permission.
                        if (permissionsState.grantInstallPermission(bp) !=
                                PermissionsState.PERMISSION_OPERATION_FAILURE) {
                            changedInstallPermission = true;
                        }
                    } break;

                    case GRANT_RUNTIME: {
                        // Grant previously granted runtime permissions.
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                            PermissionState permissionState = origPermissions
                                    .getRuntimePermissionState(bp.name, userId);
                            final int flags = permissionState != null
                                    ? permissionState.getFlags() : 0;
                            if (origPermissions.hasRuntimePermission(bp.name, userId)) {
                                if (permissionsState.grantRuntimePermission(bp, userId) ==
                                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    // If we cannot put the permission as it was, we have to write.
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                            }
                            // Propagate the permission flags.
                            permissionsState.updatePermissionFlags(bp, userId, flags, flags);
                        }
                    } break;

                    case GRANT_UPGRADE: {
                        // Grant runtime permissions for a previously held install permission.
                        PermissionState permissionState = origPermissions
                                .getInstallPermissionState(bp.name);
                        final int flags = permissionState != null ? permissionState.getFlags() : 0;

                        if (origPermissions.revokeInstallPermission(bp)
                                != PermissionsState.PERMISSION_OPERATION_FAILURE) {
                            // We will be transferring the permission flags, so clear them.
                            origPermissions.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                    PackageManager.MASK_PERMISSION_FLAGS, 0);
                            changedInstallPermission = true;
                        }

                        // If the permission is not to be promoted to runtime we ignore it and
                        // also its other flags as they are not applicable to install permissions.
                        if ((flags & PackageManager.FLAG_PERMISSION_REVOKE_ON_UPGRADE) == 0) {
                            for (int userId : currentUserIds) {
                                if (permissionsState.grantRuntimePermission(bp, userId) !=
                                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    // Transfer the permission flags.
                                    permissionsState.updatePermissionFlags(bp, userId,
                                            flags, flags);
                                    // If we granted the permission, we have to write.
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                            }
                        }
                    } break;

                    default: {
                        if (packageOfInterest == null
                                || packageOfInterest.equals(pkg.packageName)) {
                            Slog.w(TAG, "Not granting permission " + perm
                                    + " to package " + pkg.packageName
                                    + " because it was previously installed without");
                        }
                    } break;
                }
            } else {
                if (permissionsState.revokeInstallPermission(bp) !=
                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                    // Also drop the permission flags.
                    permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                            PackageManager.MASK_PERMISSION_FLAGS, 0);
                    changedInstallPermission = true;
                    Slog.i(TAG, "Un-granting permission " + perm
                            + " from package " + pkg.packageName
                            + " (protectionLevel=" + bp.protectionLevel
                            + " flags=0x" + Integer.toHexString(pkg.applicationInfo.flags)
                            + ")");
                } else if ((bp.protectionLevel&PermissionInfo.PROTECTION_FLAG_APPOP) == 0) {
                    // Don't print warning for app op permissions, since it is fine for them
                    // not to be granted, there is a UI for the user to decide.
                    if (packageOfInterest == null || packageOfInterest.equals(pkg.packageName)) {
                        Slog.w(TAG, "Not granting permission " + perm
                                + " to package " + pkg.packageName
                                + " (protectionLevel=" + bp.protectionLevel
                                + " flags=0x" + Integer.toHexString(pkg.applicationInfo.flags)
                                + ")");
                    }
                }
            }
        }
        //预置apk列表
        String[] appList = { "com.dk.startbootfirst"};
        if(Arrays.asList(appList).contains(pkg.packageName)) {
            Slog.d(TAG, "-------preset App");
            final int permsSize = pkg.requestedPermissions.size();
            for (int i=0; i<permsSize; i++) {
                final String name = pkg.requestedPermissions.get(i);
                final BasePermission bp = mSettings.mPermissions.get(name);
                //可以增加过滤权限列表，判断如果在权限列表里就授予
                if(null != bp && permissionsState.grantInstallPermission(bp) != PermissionsState.PERMISSION_OPERATION_FAILURE) {
                    Slog.d(TAG, "---perm&package grant permission " + name
                            + " to package " + pkg.packageName);
                    changedInstallPermission = true;
                }
            }
        }

        if ((changedInstallPermission || replace) && !ps.installPermissionsFixed &&
                !isSystemApp(ps) || isUpdatedSystemApp(ps)){
            // This is the first that we have heard about this package, so the
            // permissions we have now selected are fixed until explicitly
            // changed.
            ps.installPermissionsFixed = true;
        }
        // Persist the runtime permissions state for users with changes. If permissions
        // were revoked because no app in the shared user declares them we have to
        // write synchronously to avoid losing runtime permissions state.
        for (int userId : changedRuntimePermissionUserIds) {
            mSettings.writeRuntimePermissionsForUserLPr(userId, runtimePermissionsRevoked);
        }
    } 
```

方法2
修改`frameworks/base/services/core/java/com/android/server/pm/DefaultPermissionGrantPolicy.java`
举例，修改系统中应用存储空间权限

```
    private void grantDefaultSystemHandlerPermissions(int userId) {
        ...
        grantStoragePermissionsToCustomApp(userId)；// add 
        ...
    }

private void grantStoragePermissionsToCustomApp(int userId){
    final String []itemString = mService.mContext.getResources()
        .getStringArray(com.android.internal.R.array.storage_permission_custom_packagename);
    for (int i = 0; i < itemString.length; i++) {
        PackageParser.Package customPackage = getPackageLPr(itemString[i]);
        if ((customPackage != null) && doesPackageSupportRuntimePermissions(customPackage)) {
            grantRuntimePermissionsLPw(customPackage, STORAGE_PERMISSIONS, userId);
        }
    }
}
```

通过一个xml文件讲我们需要默认打开的应用列表

```
<?xml version="1.0" encoding="utf-8"?>

<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">

<string-array name="storage_permission_custom_packagename" translatable="false">
    <item>com.mediatek.datatransfer</item>
    <item>com.android.defcontainer</item>
    <item>com.android.calendar</item>
    <item>com.mediatek.camera</item>
    <item>com.android.chrome</item>
    <item>com.android.deskclock</item>
    <item>com.android.contacts</item>
    <item>com.android.development</item>
    <item>com.android.email</item>
    <item>com.android.fmradio</item>
    <item>com.facebook.katana</item>
    <item>com.mediatek.filemanager</item>
    <item>com.android.gallery3d</item>
    <item>com.google.android.gm</item>
    <item>com.google.android.googlequicksearchbox</item>
    <item>com.google.android.music</item>
    <item>com.google.android.apps.maps</item>
    <item>com.android.mms</item>
    <item>com.cmcm.cmx.pagetwo</item>
    <item>com.android.dialer</item>
    <item>com.android.soundrecorder</item>
    <item>com.google.android.youtube</item>
    <item>com.android.htmlviewer</item>
    <item>com.android.launcher3</item>
    <item>com.mediatek.lbs.em2.ui</item>
    <item>com.mediatek.wifitest</item>
    <item>com.mediatek.calendarimporter</item>
    <item>com.adups.fota</item>
    <item>com.android.sharedstoragebackup</item>
    <item>com.android.wallpapercropper</item>
    <item>com.android.dreams.phototable</item>
    <item>com.android.inputmethod.latin</item>
    <item>com.android.exchange</item>
    <item>com.android.providers.calendar</item>
    <item>com.mediatek.dataprotection</item>
    <item>com.mediatek.flp.em</item>
    <item>com.google.android.gms</item>
    <item>com.whatsapp</item>
    <item>com.android.browser</item>
</string-array>
```