## How to build ROM LineageOS 20 (Android 13) for S7, S7E, S8, S8 PLUS, NOTE 8

```markdown
repo init -u https://github.com/LineageOS/android.git -b lineage-20.0 --git-lfs
```
**Copy local_manifests to .repo** 

```markdown
repo sync -c -j16 --force-sync --no-clone-bundle --no-tags
```


# Auto Boot :
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

    
# Enable ADB && Root : 
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
    
    
# Add GApps :
```markdown
git clone https://gitlab.com/MindTheGapps/vendor_gapps/-/blob/tau/arm64/Android.bp?ref_type=heads
```

- copy to /vendor/gapps

```markdown
nano device/samsung/greatlte/device.mk : $(call inherit-product, vendor/gapps/arm64/arm64-vendor.mk)
```


# Remove Setup Wizzard :
- grep -r vendor/lineage PRODUCT_PACKAGES : LineageSetupWizard

- grep -r /vendor/gapps PRODUCT_PACKAGES : SetupWizard

- vendor/gapps/arm64/Android.bp : remove Setup Wizzard in last line

    

# Auto grant Perrmission for app : 

Edit packages/apps/Settings/src/com/android/settings/applications/appinfo

+ ExternalSourcesDetails.java :
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
+ ManageExternalStorageDetails.java :
    ```markdown
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


 

