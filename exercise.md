# Android App Repackaging Attendee Exercise

## Safety and scope

Use this exercise only on apps you own, control, or are explicitly authorized to test.
Do not test production systems or third-party accounts without permission.

## Learning goals

By the end of this lab, you will be able to:

- Configure an Android emulator with Google Play
- Install and use reverse engineering and instrumentation tools
- Extract base and split APK files from a device/emulator
- Instrument an app with ACVTool and generate coverage data
- Inspect code in JADX with ACV context
- Patch smali to add Logcat logging for selected values
- Rebuild, sign, install, and validate behavior


## 1) Emulator setup (with Google Play)

1. Install Android Studio.
2. Open Device Manager and create an AVD.
3. Choose a system image that explicitly includes Play Store (Google Play icon shown in AVD list).
4. Start emulator.
5. Sign in to Google Play.

Account guidance:

- Prefer creating a dedicated workshop Google account (recommended)
- If using personal account, avoid syncing private data

## 2) Install required tools (macOS)

Install base tools:

- Android Studio
- Apktool
- JaDX

Install ACVTool and related Python deps:

```bash
python3 -m pip install --upgrade pip
python3 -m pip install acvtool
```

Install acvpatcher (dependency of ACVTool) from here https://github.com/pilgun/acvpatcher

Optional signer (useful when you need explicit signing steps):

Uber APK Signer (JAR): https://github.com/patrickfav/uber-apk-signer

Validate tool availability:

```bash
adb version
apktool --version
jadx --version || jadx-gui --help
acv --help
acvpatcher --help
smali --help || true
baksmali --help || true
```

Notes:

- In many setups, smali/baksmali are available via ACVTool environment or separate install.
- If smali or baksmali are missing, install them separately and re-check PATH.

## 3) Verify emulator connectivity

```bash
adb devices
```

Expected: one emulator device in state device.

## 4) Install or pick a target app

Option A: Install from Play Store directly on emulator.
Option B: Use an app already installed on emulator/device.

List installed packages:

```bash
adb shell pm list packages -f -3
```

Find your target package quickly:

```bash
adb shell pm list packages | grep -i "yourapp"
```

Set a package variable:


## 5) Pull base APK and split APKs

Create workspace and list APK paths:

```bash
adb shell pm path "yourpackage" | sed 's/^package://'
```

Pull all APK artifacts:

```bash
adb pull /data/app/~~*****==/******==/base.apk ./
adb pull /data/app/~~*****==/******==/split_config.xxhdpi.apk ./
...
```

You should now have base.apk and possibly multiple split_config*.apk files.

## 6) Instrument with ACVTool

Go to pulled APK folder:

Instrument base APK:

```bash
acv instrument base.apk
cp ~/acvtool/acvtool_working_dir/intr_package.apk ./
rm base.apk
```

Patch/sign all APK files before install:

```bash
for file in ./*.apk; do acvpatcher -a $file; done
```

Install instrumented split set:

```bash
adb install-multiple ./*.apk
```

If install fails due to signatures/version conflicts, uninstall target and retry:

```bash
adb uninstall "PKG"
adb install-multiple ./*.apk
```

## 7) Run app and collect coverage

1. Launch app in emulator.
2. Execute a few user actions (navigate screens, create/edit items, search, etc.).
3. Return to terminal and collect coverage:



```bash
acv activate com.package.android
acv snap com.package.android
acv cover-pickles com.package.android
acv report com.package.android -json -shrink
```

## 8) Inspect code with JADX (and ACV context)

Open app in JADX GUI:

```bash
jadx-gui base.apk
```

If using ACV/JADX plugin:

1. Install plugin according to your local setup.
2. Reopen target APK in JADX.
3. Navigate classes/methods and compare with coverage outputs.

Goal: Find an interesting method where observing runtime values helps understanding behavior.

## 9) Add a small smali logging patch

Pick one method and add safe Logcat logging for a non-sensitive value (for workshop demo).

Disassemble target dex:

```bash
unzip -o base.apk classes.dex
baksmali d classes.dex -o classes
```

Edit chosen smali method and inject one log call, for example:

```smali
const-string v0, "LAB"
const-string v1, "Reached method X"
invoke-static {v0, v1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
```

Rebuild dex and repack:

```bash
smali a classes -o classes.dex
acvpatcher -a base.apk -c classes.dex
```

Reinstall full APK set:

```bash
adb install-multiple ./*.apk
```

Observe logs while exercising the app:

```bash
adb logcat | grep "LAB\|PATCH"
```

## 10) Validate results

Checklist:

- App launches after instrumentation/repackaging
- Coverage output is produced
- JADX navigation works for target classes
- Smali patch executes and appears in Logcat

## Common troubleshooting

- INSTALL_FAILED_UPDATE_INCOMPATIBLE
  - Uninstall app before reinstalling patched APKs.
- Split APK mismatch
  - Ensure base.apk and all split_config*.apk files are installed together.
- Missing tools in PATH
  - Reopen shell or export PATH to Python user bin.
- Crashes after patch
  - Revert patch and add smaller logging statement with correct register usage.

## Suggested workshop timing (90 minutes)

- 20 min: emulator + tooling setup
- 20 min: package discovery + APK extraction
- 20 min: ACV instrumentation + coverage collection
- 20 min: JADX review + smali patch + reinstall
- 10 min: wrap-up and Q&A

