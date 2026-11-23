# main 函数启动流程函数清单

> 依据当前分支 `feature-yaoming-remote` 源码整理，按代码实际执行顺序排列。
> 本表只展开到 `EGUIMain()` 直接调用的第一层；`InitGui()`、`StartDvr()`、`MPU_Init()`、`MPU_Init2()` 等内部还有大量二级初始化。

## 一、入口链路

| 顺序 | 位置 | 函数 | 说明 |
| --- | --- | --- | --- |
| 1 | `code/app/GUI_8x/gui_main.c:8163` | `__wrap_main` | 链接器包装的 main，直接转调 `my_ecos_main` |
| 2 | `code/midware/EGUI/src/main/main.c:54` | `my_ecos_main` | 程序真正入口 |
| 3 | `code/midware/EGUI/src/main/main.c:63` | `InitEGUI` | EGUI 内核初始化，失败则返回 1 |
| 4 | `code/midware/EGUI/src/main/main.c:73` | `EGUIMain` | 应用主流程（实现在 `gui_main.c:9326`），见下文 |
| 5 | `code/midware/EGUI/src/main/main.c:77` | `TerminateEGUI` | 主流程退出后终止 EGUI |

```c
// main.c:54
int my_ecos_main(int args, const char *agr[])
{
    int iRet = 0;
    if (!InitEGUI()) { ... return 1; }
    iRet = EGUIMain(args, agr);
    TerminateEGUI(iRet);
    return iRet;
}
```

## 二、EGUIMain 启动主流程

以下行号均指 `code/app/GUI_8x/gui_main.c`。

### 阶段 A：基础环境（9329–9411）

| # | 行号 | 函数 / 调用 | 说明 | 编译开关 |
| --- | --- | --- | --- | --- |
| 1 | 9330 | `zram_swap_init("100M", 100, 512, 200)` | ZRAM 交换分区初始化 | `_ENABLE_ZRAM_SWAP_` |
| 2 | 9334 | `handle_signal_init()` | 初始化信号处理 | |
| 3 | 9343 | `memset(&tGuiWnd, 0, ...)` | 清空全局窗口句柄 `g_tGuiWnd` | |
| 4 | 9346 | `MPU_SYS_Buzzer(BUZZER500)` | 程序启动响一声蜂鸣器 | |
| 5 | 9352 | `InitGui()` | 载入图片、初始化语言、软键盘、附加库 | |
| 6 | 9355 | `MPU_PUB_GetDeviceCap(&tDC)` | 获取设备能力 | |
| 7 | 9358 | `MPU_AVL_GetChNum()` | 取实际编码通道数 | |
| 8 | 9359 | `MPU_NET_GetCloseModify()` | 取智能修改开关 | |
| 9 | 9362 | `MPU_PUB_GetMuteStatus()` | 加载静音状态 | |
| 10 | 9363 | `MP4Sys_EncSetMute(...)` | 将静音状态设置到编码器 | |
| 11 | 9366 | `MPU_PUB_GetVolume()` | 从配置取开机音量 | |
| 12 | 9367 | `MP4Sys_SetVolume(iVolume)` | 设置音量到 SDK | |
| 13 | 9374 | `MPU_NET_GetPushCheckInterval(...)` | 基础信息日志推送间隔（上电首次 24 小时） | |
| 14 | 9378 | `Main_ResolvingRectChange()` | 初始化界面分辨率、尺寸、字体信息 | |
| 15 | 9382 | `SetDesktopOut(...)` | 设置显示设备及分辨率 | |
| 16 | 9383 | `FB_SetHIFBShowSt(ON)` | 打开 HIFB 显示 | |
| 17 | 9386 | `MPU_NET_GetP2pShow()` | 是否在桌面显示云状态 | |
| 18 | 9388 | `TLog_Init()` | 远程日志模块初始化 | |
| 19 | 9391 | `initlanswitch(MPU_NET_GetLanPorts10M())` | 网口交换初始化 | `_ENABLE_LAN_SWITCH_` |
| 20 | 9392 | `MPU_NET_GetVLanClose()` / `setVlanClose(1)` | 查询并关闭 VLAN | `_ENABLE_LAN_SWITCH_` |
| 21 | 9399 | `GuiModuleRegCmd()` | 注册日志模块命令 | |
| 22 | 9402 | `SetDvrLanguage()` | 设置语言 | |
| 23 | 9405 | `CreateGuiDesktop()` | 创建桌面窗口并铺满透明小块 | |
| 24 | 9408 | `CreateStartWnd(HWND_DESKTOP, ...)` | 创建开关机画面窗口，同时赋给 `g_hPopWnd` | |
| 25 | 9411 | `ShowStartProc(1, _T(496), 1)` | 开机进度 1：正在初始化编解码系统 | |

### 阶段 B：SDK 回调注册（9414–9429）

| # | 行号 | 函数 / 调用 | 说明 |
| --- | --- | --- | --- |
| 26 | 9414 | `MPU_FIU_SetUiCallback(UpdateFrontStatus)` | 前端设备连接状态检测回调 |
| 27 | 9417 | `MPU_CIU_SetuiCallback(RemoteGetGuiCallBack, RemoteSetGuiCallBack)` | 远程端获取/设置 GUI 操作回调 |
| 28 | 9420 | `MPU_FIU_SetIpcParaReportCallback(UpdateIpcParam)` | IPC 参数被动更新回调 |
| 29 | 9423 | `MPU_FIU_SetReportIpcParaSendUC(MPU_CIU_GetIPCParameterResultNotify)` | IPC 参数上报 UC 回调 |
| 30 | 9426 | `MPU_FIU_SetRepairtIpcGetIpCB(MPU_NET_GetRandomIP)` | IPC 获取随机 IP 回调 |
| 31 | 9429 | `MPU_PUB_SetSnCallback(gui_SnCallBackProc)` | 序列号 / 激活相关回调 |

### 阶段 C：可选功能模块初始化（9431–9447）

| # | 行号 | 函数 / 调用 | 说明 | 编译开关 |
| --- | --- | --- | --- | --- |
| 32 | 9433 | `ChSetMgr_Init()` | 通道参数配置管理模块初始化 | `AUTO_SET_CHANNEL_CONFIG` |
| 33 | 9438 | `ThreadCreate_FreeCached()` | 创建释放 BUFF/Cache 处理线程 | `AUTO_FREE_CACHED` |
| 34 | 9442 | `PrjMode_Init()` | 工程模式初始化 | `PROJECT_MODE` |
| 35 | 9446 | `PrevMode_Init()` | 预览模式初始化 | `PREV_MODE` |

### 阶段 D：MPU 初始化与显示参数（9450–9484）

| # | 行号 | 函数 / 调用 | 说明 | 编译开关 |
| --- | --- | --- | --- | --- |
| 36 | 9450 | `MPU_Init()` | 各模块 MPU 初始化（第一段） | |
| 37 | 9452 | `MPU_PRW_GetDeskTimeShow()` | 桌面时间显示配置 | |
| 38 | 9453 | `MPU_PRW_GetChannelOsdShow()` | 通道 OSD 显示配置 | |
| 39 | 9454 | `MPU_PRW_GetChannelNameShow()` | 通道名显示配置 | |
| 40 | 9455 | `MPU_PUB_GetShowNVRStatus()` | 桌面 NVR 状态显示配置 | |
| 41 | 9459 | `OutputAdjustWnd_InitLoadParam(GUI_MONITOR)` | 加载输出调节参数，解决开机颜色跳变 | |
| 42 | 9462 | `MPU_PRW_SetPreviewAuthorityCb(GetChannelPrivilegeCb)` | 预览权限回调，用于刷新辅屏背景 | `_SUPPORT_OUTPUTMODE_HETEROLOGOUS_` |
| 43 | 9465 | `ConfigHeterologousLayout_StartupInit()` | 加载并下发副屏相关配置 | `_SUPPORT_OUTPUTMODE_HETEROLOGOUS_` |
| 44 | 9469 | `MPU_SetDecodeStatusUiCB(Osd_FBUpdateDecStatus, NULL)` | 解码器状态回调 | |
| 45 | 9471 | `GUI_MAYBEERROR(...)` | 检查 `MPU_Init` 返回值（宏） | |
| 46 | 9474 | `SwitchLogHandle()` | 开关机日志记录 | |
| 47 | 9479 | `MPU_PUB_GetChBkgSloganParam(&stChBkgSloganPara)` | 获取通道标语叠加参数 | |
| 48 | 9480 | `ChannelBkgSloganSetDlg_SloganOverlayBkg(&g_tGuiBmp.hChannelbkg, ...)` | 将通道标语叠加到通道背景图 | |
| 49 | 9481 | `ChannelBkgSloganSetDlg_SloganOverlayBkg(&g_tGuiBmp.hChannelLock, ...)` | 将标语叠加到加锁背景图 | |
| 50 | 9483 | `ChannelBkgSloganSetDlg_ConvertChannelPngToJpg(...)` | 通道背景 PNG 转 JPG | `_SUPPORT_OUTPUTMODE_HETEROLOGOUS_` |

### 阶段 E：预览、存储与第二段 MPU 初始化（9487–9556）

| # | 行号 | 函数 / 调用 | 说明 | 编译开关 |
| --- | --- | --- | --- | --- |
| 51 | 9487 | `GUI_StartPreview()` | 开启预览 | |
| 52 | 9490 | `Prv_EnPreViewRight()` | 按当前用户权限刷新预览界面 | |
| 53 | 9494 | `ShowStartProc(1, _T(497), 2)` | 开机进度 2：正在初始化存储系统 | |
| 54 | 9496 | `SetNewDiskCallBack()` | 新硬盘回调 | |
| 55 | 9498 | `SetUsbHotPlugCallBack()` | USB 热插拔通知回调 | |
| 56 | 9501 | `GuiSetProductTestCallBack()` | 产测界面回调 | |
| 57 | 9504 | `MPU_STR_DiskInit_SetGUICallBack(ShowStartDiskInitProgress)` | 硬盘初始化进度显示回调 | |
| 58 | 9507 | `ThreadCreate_CheckDiskInitTimeOut()` | 创建硬盘初始化超时检测线程 | |
| 59 | 9510 | `MPU_Init2()` | 各模块 MPU 初始化（第二段） | |
| 60 | 9512 | `MPU_IsVrsEnabled()` | 判断 VRS 是否启用 | `SUPPORT_VRS` |
| 61 | 9517 | `MPU_SYS_Model_GetInstalledStatus(...)` | 检查模型是否完整/兼容 | `SUPPORT_VRS` |
| 62 | 9520 | `VRS_RuntimeInit()` | VRS 运行时初始化 | `SUPPORT_VRS` |
| 63 | 9535 | `rtsptcpserverinit(...)` | 历史 RTSP 拉流服务初始化 | `_SUPPORT_HISTORY_RTSP_PULL_` |
| 64 | 9548 | `ThreadCreate_NotifyCIU()` | 创建 CIU 通知线程 | |
| 65 | 9550 | `DeviceSystemSpecialMaintenance_Data_Init()` | 设备特殊维护数据初始化 | |
| 66 | 9552 | `CreateStartWnd(g_tGuiWnd.hDeskWnd, ...)` | 重新创建开机画面窗口 | |
| 67 | 9553 | `ShowWindow(g_hPopWnd, SW_HIDE)` | 隐藏开机画面窗口 | |
| 68 | 9556 | `MPU_SYS_Maintenance_SetCallBack(Maintenance_Cb)` | 维护回调 | |

### 阶段 F：主界面与业务启动（9559–9601）

| # | 行号 | 函数 / 调用 | 说明 | 编译开关 |
| --- | --- | --- | --- | --- |
| 69 | 9559 | `ShowStartProc(1, _T(498), 3)` | 开机进度 3：正在加载位图资源 | |
| 70 | 9562 | `ShowStartProc(1, _T(499), 4)` | 开机进度 4：正在启动软件 | |
| 71 | 9565 | `SetUboot_MacSnVersion()` | 把 MAC/SN/VERSION 写入 U-Boot 参数区 | |
| 72 | 9568 | `CreatWnd()` | 创建桌面主窗口 | |
| 73 | 9571 | `WizzardWnd_Show(0)` | 初始隐藏开机向导窗口 | |
| 74 | 9574 | `StartDvr()` | 启动 DVR 业务 | |
| 75 | 9576 | `ConfigSysinfoRtspLicense_StartAsyncCache()` | 异步缓存 RTSP 授权信息 | `_SUPPORT_HISTORY_RTSP_PULL_` |
| 76 | 9579 | `ShowStartProc(0, "over", 5)` | 开机进度结束 | |
| 77 | 9584 | `ModelUpgrade_StartupNotify(g_tGuiWnd.hDeskWnd)` | 模型升级启动提示 | `SUPPORT_VRS` |
| 78 | 9589 | `SendNotifyMessage(GetStartWindow(), MSG_CHECKDISK, 0, 0)` | 向开机窗口发检查硬盘消息 | |
| 79 | 9591 | `SetRemoteKeyparam(..., EnablePanel_CB)` | 设置远程按键/面板参数 | |
| 80 | 9594 | `Osd_ShowRegisterState()` | 显示序列号注册状态 | |
| 81 | 9597 | `ShowDeskP2PDiskState()` | 显示云状态/硬盘状态 | |
| 82 | 9599 | `ShowDeskProjectModeTime()` | 工程模式时间显示 | `PROJECT_MODE` |

### 阶段 G：后台线程与收尾初始化（9603–9642）

| # | 行号 | 函数 / 调用 | 说明 | 编译开关 |
| --- | --- | --- | --- | --- |
| 83 | 9603 | `SrcdevMgr_Init(Srcdev_EventNotify)` | 创建自动通道连接线程 | |
| 84 | 9607 | `ChSetMgr_Start()` | 启动通道配置设置线程 | `AUTO_SET_CHANNEL_CONFIG` |
| 85 | 9609 | `ThreadCreate_CheckProc()` | 创建过程检查线程 | |
| 86 | 9612 | `MP4Sys_BSupportPanelLed()` | 判断是否支持面板灯 | `PANEL_LED` |
| 87 | 9615 | `ThreadCreate_LedProc()` | 创建面板灯显示处理线程 | `PANEL_LED` |
| 88 | 9620 | `MPU_PUB_GetDeviceInfo(&tDevInfo)` | 获取设备信息 | |
| 89 | 9624 | `ConfigSysinfoSrcdev_SetLocalPage(0, TRUE)` | 设置本地通道页 | |
| 90 | 9625 | `WizzardWnd_Show(1)` | 非首次启动时打开开机向导窗口 | |
| 91 | 9631 | `ConfigStorage_SetVrsPictureLayoutGuiWindow(g_hPopWnd)` | 设置 VRS 图片布局窗口 | `SUPPORT_VRS` |
| 92 | 9636 | `LanguageCheck_UpdateLanguage()` | 检查语言包是否需要升级 | |
| 93 | 9639 | `GetSplitScreen_RefreshBack()` | 获取分屏参数并刷新通道背景 | |
| 94 | 9642 | `FmCheck_UsbUpgradeFileCheck()` | 检测 U 盘中是否存在 U-Boot 升级文件 | |
| 95 | 9646 | `AUTOTEST_PRINTK_LOG(AUTOTEST_SYSTEM_BOOT_COMPLETED)` | 自动化测试：开机完成打点（宏） | `AUTOTEST_OUT` |

### 阶段 H：消息循环与退出清理（9649–9701）

| # | 行号 | 函数 / 调用 | 说明 | 编译开关 |
| --- | --- | --- | --- | --- |
| 96 | 9649 | `GetMessage(&Msg, g_tGuiWnd.hDeskWnd)` | 主消息循环取消息 | |
| 97 | 9651 | `TranslateMessage(&Msg)` | 消息翻译 | |
| 98 | 9652 | `DispatchMessage(&Msg)` | 消息分发 | |
| 99 | 9655 | `s_GuiMainExit = 1` | 置位进程退出标志 | |
| 100 | 9658 | `ChSetMgr_Stop()` | 停止通道配置线程 | `AUTO_SET_CHANNEL_CONFIG` |
| 101 | 9661 | `PrjMode_UnInit()` | 工程模式反初始化 | `PROJECT_MODE` |
| 102 | 9665 | `SrcDevMgr_UnInit()` | 反初始化自动通道连接 | |
| 103 | 9668 | `VRS_RuntimeDeinit()` | VRS 运行时反初始化 | `SUPPORT_VRS` |
| 104 | 9675 | `UnInitGui()` | GUI 资源反初始化 | |
| 105 | 9677 | `ConfigSysinfoRtspLicense_StopAsyncCache()` | 停止 RTSP 授权异步缓存 | `_SUPPORT_HISTORY_RTSP_PULL_` |
| 106 | 9686 | `rtsptcpserveruninit()` | 关闭 RTSP 拉流服务（最多重试 200 次，每次 100ms） | `_SUPPORT_HISTORY_RTSP_PULL_` |
| 107 | 9701 | `MPU_Uninit()` | MPU 各模块反初始化 | |

## 三、时序速览

```
__wrap_main
  └─ my_ecos_main
       ├─ InitEGUI()
       ├─ EGUIMain()
       │    ├─ 基础环境：信号/ZRAM/蜂鸣器 → InitGui → 分辨率/显示/日志/语言
       │    ├─ 回调注册：FIU / CIU / SN
       │    ├─ MPU_Init() → 预览参数 → 标语背景
       │    ├─ GUI_StartPreview() → 磁盘/USB/产测回调 → MPU_Init2()
       │    │      └─ VRS / RTSP 拉流服务（按开关）
       │    ├─ CreatWnd() → StartDvr() → 各类桌面状态显示
       │    ├─ 后台线程：SrcdevMgr / ChSetMgr / CheckProc / LedProc
       │    ├─ while (GetMessage) { TranslateMessage; DispatchMessage; }
       │    └─ 退出清理：ChSetMgr_Stop → SrcDevMgr_UnInit → VRS → UnInitGui
       │             → RTSP 反初始化 → MPU_Uninit
       └─ TerminateEGUI()
```
