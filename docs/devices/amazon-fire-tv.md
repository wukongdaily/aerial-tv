# 亚马逊 Fire TV

来自本机实测。Fire TV 是这个仓库唯一一台**亚马逊自家固件**的设备，两条接管路子（无障碍守护、默认桌面）在这儿都不成立，所以单独记。

| 机型 | 系统 | 结果 |
|---|---|---|
| Fire TV Stick 4K（AFTMM，mantis） | Fire OS 6.7.1.1 / Android 7.1 | ⚠️ 能装能日常用，**主页键接管不了**：固件层拒绝绑定第三方无障碍服务。右上角设置按钮已由本 fork 修好 |
| Fire TV Stick 4K Max 2023 | Fire OS 8 / Android 11 | Aerial 的守护在这台上**没测过**（同台的另一个应用的无障碍服务能正常绑定，见下） |

## 装得上，也要授权

Android 7.1 直接装（完整版要 Android 5.0+）。三条授权和别的盒子一样：

```sh
adb connect <电视IP>:5555
adb shell pm grant app.orchard.tv android.permission.WRITE_SECURE_SETTINGS
adb shell cmd notification allow_listener app.orchard.tv/app.orchard.tv.system.RevivalListener
adb shell appops set app.orchard.tv SYSTEM_ALERT_WINDOW allow
```

**Android 7.1 要手工预编译一次**，否则第一次滑动很卡。官方 v1.72 起的「安装时就编译」需要 Android 9，7.1 上没有这个能力：

```sh
adb shell cmd package compile -m speed -f app.orchard.tv     # 几十秒，每次升级后重来
```

**从源码装的 APK 覆盖不了官方版**：签名不同，`install -r` 会失败，只能先 `adb uninstall app.orchard.tv`，个人设置（选好的壁纸、照片）会一起清掉。

## 右上角的设置按钮（本 fork 改的就是这个）

原版在 Fire TV 上点了没反应。原因是 `android.settings.SETTINGS` 在 Fire OS 上解析到
`com.amazon.tv.settings.v2/.compliance.CTSDummySettingsHandler`，一个合规用的假页面，打开就自己关掉。真正的设置主页在 launcher 包里：

```sh
adb shell am start -n com.amazon.tv.launcher/.ui.MainSettingsActivity     # 完整设置九宫格
```

`SystemState.settingsActivityIntent()` 的解析链最前面按功能检测加了这条（看这个包在不在，和 `docs/devices/` 里别的机型的写法一致）。实测从 Aerial 按设置 → 设置主页正常打开，且不会被弹回引擎当成原桌面抢回去。

## 主页键接管不了：Fire OS 6 拒绑第三方无障碍

AFTMM 上的证据：

- `settings put secure enabled_accessibility_services` 写得进去、重启也在，但 `dumpsys accessibility` 里的 services 一直是空的——系统不绑。
- Aerial 自己的诊断日志在反复求系统绑，没结果：`AerialDiag: revival: attempt 1: unlocked, guard unbound; requesting recovery`、`AerialGuard: Home Guard was enabled but not running; asked Android to bind it again`。
- 通知监听（被清理后拉起）同样绑不上。
- `cmd package set-home-activity app.orchard.tv/.MainActivity` 返回 Success 也没用，HOME 仍归原桌面；原桌面是受保护包，`pm disable-user` 被拒，也不能禁（Fire OS 的设置页就在它里面）。

所以这台需要**外部弹回引擎**。按项目根本，本仓库的安装脚本和代理说明只安装 Aerial 自己，不下载也不推荐别的 App——这里只记需要什么机制：`moveTaskToFront` 只能抬自己进程的任务，而直接 `startActivity` 拉起 Aerial 会撞 Fire OS「按 Home 后 5 秒不允许切应用」的限制。可用的路子是让弹回器先把**自己**的界面抬到前台（闹钟由 system_server 发出，免这个延迟），再以前台身份启动 Aerial，实测 20–50 毫秒；跳过这一层直接拉 Aerial，就是 5 秒。

## 汉化会把键盘弄崩（和 Aerial 无关，但一定会踩）

`settings put system system_locales zh-CN` 之后，任何输入界面都弹不出键盘：Fire TV 的键盘进程 `com.amazon.tv.ime` 在 `FireTVIME.onStartInput` 里对 `ILanguageEngine` 调 `setConfiguration` 空指针秒崩。Fire OS 6 和 Fire OS 8 两台都复现，不是固件代际差异，是这套地区固件的语言引擎没有简体中文包。改 `zh-TW` 后界面是中文、键盘正常。Aerial 的界面语言跟系统走，也跟着变中文。

顺带两条：汉化前先用 `adb shell date` 确认 NTP 已同步（时间停在固件构建日时，亚马逊服务器的异步下发会一直失败）；时区用 `adb shell cmd alarm set-timezone Asia/Shanghai`，`setprop persist.sys.timezone` 被 Fire OS 拒。

## 没验证过的

- Fire OS 8（Android 11）上 Aerial 的守护能不能绑上。
- 配上外部弹回之后，待机唤醒、深度休眠后回 Aerial 的表现。
- 4K 与杜比视界航拍影片在 Stick 4K Max 上的实际输出。
