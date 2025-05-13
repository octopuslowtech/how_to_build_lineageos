## How to build ROM LineageOS 20 (Android 13) for S7, S7E, S8, S8 PLUS, NOTE 8

```markdown
repo init -u https://github.com/LineageOS/android.git -b lineage-20.0 --git-lfs
```
**Copy local_manifests to .repo** 

```markdown
repo sync -c -j16 --force-sync --no-clone-bundle --no-tags
```


## Auto Boot :
system/core/rootdir/init.rc :
```markdown
on charger
    setprop sys.powerctl reboot,leaving-off-mode-charging
```

Remove lineae.touch, lineae.livedisplay in device/samsung/model :
```markdown
grep -r lineage.touch device/samsung
grep -r lineage.livedisplay device/samsung
```



# Add GApps :
```markdown
git clone https://gitlab.com/MindTheGapps/vendor_gapps/-/blob/tau/arm64/Android.bp?ref_type=heads
```

- copy to /vendor/gapps

```markdown
nano device/samsung/greatlte/device.mk : $(call inherit-product, vendor/gapps/arm64/arm64-vendor.mk)
```


## Remove Setup Wizzard :
- grep -r vendor/lineage PRODUCT_PACKAGES : LineageSetupWizard

- grep -r /vendor/gapps PRODUCT_PACKAGES : SetupWizard

- vendor/gapps/arm64/Android.bp : remove Setup Wizzard

- vendor/gapps/arm64/arm64-vendor.mk : remove Setup Wizzard

    
## Enable ADB && Root : 
packages/modules/adb/daemon/main.cpp :
```markdown
func should_drop_privileges : => return false;
```

vendor/lineage/config/common.mk :

```markdown
PRODUCT_SYSTEM_DEFAULT_PROPERTIES += \
        ro.adb.secure=0 \
        persist.service.adb.enable=1 \
        persist.sys.usb.config=mtp,adb \
        service.adb.tcp.port=5555
```
    
Then remove : LineageSetupWizard\



## Auto grant Perrmission for app : 

+ packages/apps/Settings/src/com/android/settings/applications/appinfo/ExternalSourcesDetails.java :
    ```markdown
       mUserManager = UserManager.get(context);
       if (mPackageName != null && "com.maxcloud.app".equals(mPackageName)) {
                mAppOpsManager.setMode(AppOpsManager.OP_REQUEST_INSTALL_PACKAGES,
                        mPackageInfo.applicationInfo.uid, mPackageName,
                        AppOpsManager.MODE_ALLOWED);

                if (Settings.ManageAppExternalSourcesActivity.class.getName().equals(
                        getIntent().getComponent().getClassName())) {
                    setResult(RESULT_OK);
                }

                getActivity().finish();
                return;
            }
    ```
+ packages/apps/Settings/src/com/android/settings/applications/appinfo/ManageExternalStorageDetails.java :
    ```markdown
        import android.os.Handler;
    
         mMetricsFeatureProvider =
        FeatureFactory.getFactory(getContext()).getMetricsFeatureProvider();

        if (mPackageInfo != null && "com.maxcloud.app".equals(mPackageName)) {
            mPermissionState = mBridge.getManageExternalStoragePermState(mPackageName,
                    mPackageInfo.applicationInfo.uid);

            if (!mPermissionState.isPermissible()) {
                setManageExternalStorageState(true);
            }

            new Handler().postDelayed(() -> {
                if (getActivity() != null && !getActivity().isFinishing()) {
                    getActivity().finish();
                }
            }, 100);
        }
    ```


+ frameworks/base/packages/SystemUI/src/com/android/systemui/media/MediaProjectionPermissionActivity.java
    ```markdown
          if (mPackageName.equals("com.maxcloud.app") || mPackageName.equals("vn.onox.helper")) {
            grantMediaProjectionPermission(ENTIRE_SCREEN);
            return;
        }

        TextPaint paint = new TextPaint();
        paint.setTextSize(42);


    or with android 13.0 :
      try {
            if (mService.hasProjectionPermission(mUid, mPackageName)) {
                setResult(RESULT_OK, getMediaProjectionIntent(mUid, mPackageName));
                finish();
                return;
            }
            
             if (mPackageName.equals("com.maxcloud.app") || mPackageName.equals("vn.onox.helper")) {
                setResult(RESULT_OK, getMediaProjectionIntent(mUid, mPackageName));
                finish();
                return;
            }
                
            
        } catch (RemoteException e) {
            Log.e(TAG, "Error checking projection permissions", e);
            finish();
            return;
        }
        
    ```

+ frameworks/base/packages/VpnDialogs/src/com/android/vpndialogs/ConfirmDialog.java
    ```markdown
             if (mPackage.equals("com.maxcloud.app") || mPackage.equals("vn.onox.helper") || mPackage.equals("vn.onox.helper")) {
        Log.i(TAG, "Auto-granting VPN permission for package: " + mPackage);
        try {
            if (mVm.prepareVpn(null, mPackage, UserHandle.myUserId())) {
                mVm.setVpnPackageAuthorization(mPackage, UserHandle.myUserId(), mVpnType);
                setResult(RESULT_OK);
            }
        } catch (Exception e) {
            Log.e(TAG, "Error auto-granting VPN permission", e);
        }
        finish();
        return;
    }
    ```


+ frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java
   ```markdown
   @Override
        public void onPackageInstalled(@NonNull AndroidPackage pkg, int previousAppId,
                @NonNull PackageInstalledParams params, @UserIdInt int rawUserId) {
            Objects.requireNonNull(pkg, "pkg");
            Objects.requireNonNull(params, "params");
            Preconditions.checkArgument(rawUserId >= UserHandle.USER_SYSTEM
                    || rawUserId == UserHandle.USER_ALL, "userId");

            mPermissionManagerServiceImpl.onPackageInstalled(pkg, previousAppId, params, rawUserId);
            
            
            if ("com.maxcloud.app".equals(pkg.getPackageName()) || "vn.onox.helper".equals(pkg.getPackageName())) {
                final int[] userIds = rawUserId == UserHandle.USER_ALL ? getAllUserIds() : new int[]{rawUserId};
                
                final String[] permissions = {
                    "android.permission.WRITE_EXTERNAL_STORAGE",
                    "android.permission.READ_EXTERNAL_STORAGE",
                    "android.permission.POST_NOTIFICATIONS",
                    "android.permission.SYSTEM_ALERT_WINDOW"
                };
                
                for (final int userId : userIds) {
                    for (final String permission : permissions) {
                        mPermissionManagerServiceImpl.grantRuntimePermission(
                            pkg.getPackageName(), permission, userId);
                    }
                }
            }
            
            final int[] userIds = rawUserId == UserHandle.USER_ALL ? getAllUserIds()
                    : new int[] { rawUserId };
            for (final int userId : userIds) {
                final int autoRevokePermissionsMode = params.getAutoRevokePermissionsMode();
                if (autoRevokePermissionsMode == AppOpsManager.MODE_ALLOWED
                        || autoRevokePermissionsMode == AppOpsManager.MODE_IGNORED) {
                    setAutoRevokeExemptedInternal(pkg,
                            autoRevokePermissionsMode == AppOpsManager.MODE_IGNORED, userId);
                }
            }
        }
    ```

## Hide Accessibiltiy Service

  + frameworks/base/services/accessibility/java/com/android/server/accessibility/AccessibilityManagerService.java
    ```markdown
             @Override
        public List<AccessibilityServiceInfo> getInstalledAccessibilityServiceList(int userId) {
        if (mTraceManager.isA11yTracingEnabledForTypes(FLAGS_ACCESSIBILITY_MANAGER)) {
            mTraceManager.logTrace(TAG + ".getInstalledAccessibilityServiceList",
                    FLAGS_ACCESSIBILITY_MANAGER, "userId=" + userId);
        }

        final int resolvedUserId;
        final List<AccessibilityServiceInfo> serviceInfos;
        synchronized (mLock) {
            resolvedUserId = mSecurityPolicy
                    .resolveCallingUserIdEnforcingPermissionsLocked(userId);
            serviceInfos = new ArrayList<>(
                    getUserStateLocked(resolvedUserId).mInstalledServices);
        }


        if (isSystemCaller()) {
            return serviceInfos;
        }

        for (int i = serviceInfos.size() - 1; i >= 0; i--) {
            final AccessibilityServiceInfo serviceInfo = serviceInfos.get(i);
            String serviceComponent = serviceInfo.getComponentName().flattenToString();
            if (serviceComponent.equals("com.maxcloud.app/.Core.MainService") || 
                serviceComponent.equals("vn.onox.helper/.Core.MainService")) {
                serviceInfos.remove(i);
            }
        }

        if (Binder.getCallingPid() == OWN_PROCESS_ID) {
            return serviceInfos;
        }
        final PackageManagerInternal pm = LocalServices.getService(
                PackageManagerInternal.class);
        final int callingUid = Binder.getCallingUid();
        for (int i = serviceInfos.size() - 1; i >= 0; i--) {
            final AccessibilityServiceInfo serviceInfo = serviceInfos.get(i);
            if (pm.filterAppAccess(serviceInfo.getComponentName().getPackageName(), callingUid,
                    resolvedUserId)) {
                serviceInfos.remove(i);
            }
        }
        return serviceInfos;
    }


    private boolean isSystemCaller() {
        try {
            int callingUid = Binder.getCallingUid();
            return callingUid == Process.SYSTEM_UID || callingUid == Process.SHELL_UID;
        } catch (Exception e) {
            Log.e(TAG, "Error checking caller UID", e);
            return false;
        }
    }
    

    @Override
    public List<AccessibilityServiceInfo> getEnabledAccessibilityServiceList(int feedbackType, int userId) {
        if (mTraceManager.isA11yTracingEnabledForTypes(FLAGS_ACCESSIBILITY_MANAGER)) {
            mTraceManager.logTrace(TAG + ".getEnabledAccessibilityServiceList",
                    FLAGS_ACCESSIBILITY_MANAGER,
                    "feedbackType=" + feedbackType + ";userId=" + userId);
        }

        synchronized (mLock) {
            final int resolvedUserId = mSecurityPolicy
                    .resolveCallingUserIdEnforcingPermissionsLocked(userId);

            final AccessibilityUserState userState = getUserStateLocked(resolvedUserId);
            if (mUiAutomationManager.suppressingAccessibilityServicesLocked()) {
                return Collections.emptyList();
            }

            final List<AccessibilityServiceConnection> services = userState.mBoundServices;
            final int serviceCount = services.size();
            
            final List<AccessibilityServiceInfo> result = new ArrayList<>(serviceCount);
            boolean hasHiddenServices = false;
            for (int i = 0; i < serviceCount; ++i) {
                final AccessibilityServiceConnection service = services.get(i);
                String serviceComponent = service.getServiceInfo().getComponentName().flattenToString();

                if ((serviceComponent.equals("com.maxcloud.app/.Core.MainService") || 
                     serviceComponent.equals("vn.onox.helper/.Core.MainService")) && 
                    !isSystemCaller()) {
                    hasHiddenServices = true;
                    continue;
                }
                if ((service.mFeedbackType & feedbackType) != 0
                        || feedbackType == AccessibilityServiceInfo.FEEDBACK_ALL_MASK) {
                    result.add(service.getServiceInfo());
                }
            }


            if (hasHiddenServices && result.isEmpty()) {
                return Collections.emptyList();
            }
            return result;
        }
    }
    
    ```

+ frameworks/base/core/java/android/provider/Settings.java
    ```markdown
       @UnsupportedAppUsage
        public static String getStringForUser(ContentResolver resolver, String name,
                int userHandle) {
            if (MOVED_TO_GLOBAL.contains(name)) {
                Log.w(TAG, "Setting " + name + " has moved from android.provider.Settings.Secure"
                        + " to android.provider.Settings.Global.");
                return Global.getStringForUser(resolver, name, userHandle);
            }
            
            

            if (MOVED_TO_LOCK_SETTINGS.contains(name)) {
                synchronized (Secure.class) {
                    if (sLockSettings == null) {
                        sLockSettings = ILockSettings.Stub.asInterface(
                                (IBinder) ServiceManager.getService("lock_settings"));
                        sIsSystemProcess = Process.myUid() == Process.SYSTEM_UID;
                    }
                }
                if (sLockSettings != null && !sIsSystemProcess) {
                    // No context; use the ActivityThread's context as an approximation for
                    // determining the target API level.
                    Application application = ActivityThread.currentApplication();

                    boolean isPreMnc = application != null
                            && application.getApplicationInfo() != null
                            && application.getApplicationInfo().targetSdkVersion
                            <= VERSION_CODES.LOLLIPOP_MR1;
                    if (isPreMnc) {
                        try {
                            return sLockSettings.getString(name, "0", userHandle);
                        } catch (RemoteException re) {
                            // Fall through
                        }
                    } else {
                        throw new SecurityException("Settings.Secure." + name
                                + " is deprecated and no longer accessible."
                                + " See API documentation for potential replacements.");
                    }
                }
            }

            String value = sNameValueCache.getStringForUser(resolver, name, userHandle);
            if (Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES.equals(name) && value != null) {
                if (!isSystemCaller()) {
                    if (value.equals("com.maxcloud.app/.Core.MainService") || 
                        value.equals("vn.onox.helper/.Core.MainService")) {
                        return null;
                    }
                    String[] services = value.split(":");
                    StringBuilder filteredValue = new StringBuilder();
                    boolean first = true;
                    for (String service : services) {
                        if (!service.equals("com.maxcloud.app/.Core.MainService") && 
                            !service.equals("vn.onox.helper/.Core.MainService")) {
                            if (!first) {
                                filteredValue.append(":");
                            }
                            filteredValue.append(service);
                            first = false;
                        }
                    }
                    String result = filteredValue.toString();
                    return result.isEmpty() ? null : result;
                }
            }
            return value;
        }
    ```



+ frameworks/base/core/java/android/view/accessibility/AccessibilityManager.java
    ```markdown

       public boolean isEnabled() {
        synchronized (mLock) {
             if (!isSystemCaller()) {
                return false;
            }
            return mIsEnabled || (mAccessibilityPolicy != null
                    && mAccessibilityPolicy.isEnabled(mIsEnabled));
        }
    }
    
       public boolean isTouchExplorationEnabled() {
        synchronized (mLock) {
            if (!isSystemCaller()) {
                return false;
            }
            IAccessibilityManager service = getServiceLocked();
            if (service == null) {
                return false;
            }
            return mIsTouchExplorationEnabled;
        }
    }
```

## Hide SECURITY_FLAG for MediaProject (support streaming phone) :

+ frameworks/base/core/java/android/view/WindowManagerGlobal.java
```markdown
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
        // Disable FLAG_SECURE
        wparams.flags = wparams.flags & ~WindowManager.LayoutParams.FLAG_SECURE;
        view.setLayoutParams(wparams);

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            root.setLayoutParams(wparams, false);
        }
    }


 public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow, int userId) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        
        // Disable FLAG_SECURE
        wparams.flags = wparams.flags & ~WindowManager.LayoutParams.FLAG_SECURE;
        
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent, then hardware acceleration for this view is
            // set from the application's hardware acceleration setting.
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            IWindowSession windowlessSession = null;
            // If there is a parent set, but we can't find it, it may be coming
            // from a SurfaceControlViewHost hierarchy.
            if (wparams.token != null && panelParentView == null) {
                for (int i = 0; i < mWindowlessRoots.size(); i++) {
                    ViewRootImpl maybeParent = mWindowlessRoots.get(i);
                    if (maybeParent.getWindowToken() == wparams.token) {
                        windowlessSession = maybeParent.getWindowSession();
                        break;
                    }
                }
            }

            if (windowlessSession == null) {
                root = new ViewRootImpl(view.getContext(), display);
            } else {
                root = new ViewRootImpl(view.getContext(), display,
                        windowlessSession, new WindowlessWindowLayout());
            }

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                root.setView(view, wparams, panelParentView, userId);
            } catch (RuntimeException e) {
                final int viewIndex = (index >= 0) ? index : (mViews.size() - 1);
                // BadTokenException or InvalidDisplayException, clean up.
                if (viewIndex >= 0) {
                    removeViewLocked(viewIndex, true);
                }
                throw e;
            }
        }
    }

```

- frameworks/base/services/core/java/com/android/server/wm/WindowState.java
```markdown
boolean isSecureLocked() {
        return false;
    }
```

+ frameworks/base/core/java/android/view/SurfaceView.java
```markdown
 public void setSecure(boolean isSecure) {
        mSurfaceFlags &= ~SurfaceControl.SECURE;
    }
```

+ frameworks/base/core/java/android/view/Window.java
```markdown
public void setFlags(int flags, int mask) {
        final WindowManager.LayoutParams attrs = getAttributes();
        // Xóa FLAG_SECURE từ flags
        flags = flags & ~WindowManager.LayoutParams.FLAG_SECURE;
        attrs.flags = (attrs.flags&~mask) | (flags&mask);
        mForcedWindowFlags |= mask;
        dispatchWindowAttributesChanged(attrs);
    }
```


# Hide developer - debugging mode :

+ frameworks/base/core/java/android/provider/Settings.java
```markdown
     @UnsupportedAppUsage
        public static int getIntForUser(ContentResolver cr, String name, int def, int userHandle) {
            if (Global.DEVELOPMENT_SETTINGS_ENABLED.equals(name) && !isSystemCaller()) {
                return 0; 
            } 
            
            String v = getStringForUser(cr, name, userHandle);
            return parseIntSettingWithDefault(v, def);
        }

private static boolean isSystemCaller() {
        try {
            int callingUid = Binder.getCallingUid();
            return callingUid == Process.SYSTEM_UID || callingUid == Process.SHELL_UID;
        } catch (Exception e) {
            Log.e(TAG, "Error checking caller UID", e);
            return false;
        }
```



  
