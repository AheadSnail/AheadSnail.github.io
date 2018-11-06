---
layout: pager
title: Fba 街机对战实现
date: 2018-10-15 09:39:21
tags: [Android,NDK,Fba]
description: Fba 街机对战实现
---

### 概述

> Fba 街机对战实现

<!--more-->

### Fba源码分析

> 前面一篇文章中分析了FBa 中引入开源的Kaillera库，可以实现街机的对战，简要的介绍了他对应的功能，也从对应的网站上下载到了开源的代码，客户端以及服务端，测试是可以使用的，下面就简要的分析 下，这个对战库在Fba 源码中是怎么样使用的，这样才能写出对应的测试代码，来调试分析对战库

前面已经分析过了，主函数的入口位置为 src/burn/win32/main.cpp文件,下面就大致的介绍看下对应的源码

```C
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE, LPSTR lpCmdLine, int nShowCmd)
{
	...
	if (!(AppInit())) {							// Init the application
		if (bAlwaysCreateSupportFolders) CreateSupportFolders();
		if (!(ProcessCmdLine())) {
			DetectWindowsVersion();
			EnableHighResolutionTiming();

			MediaInit();

			RunMessageLoop();					// Run the application message loop
		}
	}
	...
}

首先看AppInit 函数的实现
static int AppInit()
{
	...
	// Init the Burn library  
	BurnLibInit();
	...
}

extern "C" INT32 BurnLibInit()
{
	BurnLibExit();
	nBurnDrvCount = sizeof(pDriver) / sizeof(pDriver[0]);	// count available drivers

	cmc_4p_Precalc();
	bBurnUseMMX = BurnCheckMMXSupport();

	return 0;
}

这里主要是算出当前街机支持的drivers的数量,是通过 sizeof(pDriver) / sizeof(pDriver[0]),这里的pDriver 为
// Structure containing addresses of all drivers
// Needs to be kept sorted (using the full game name as the key) to prevent problems with the gamelist in Kaillera
static struct BurnDriver* pDriver[] = {
	&BurnDrvgames88,			// '88 Games
	&BurnDrvFlagrall,			// '96 Flag Rally
	&BurnDrv99lstwar,			// '99: The Last War
	&BurnDrv99lstwara,			// '99: The Last War (alternate)
	&BurnDrv99lstwark,			// '99: The Last War (Kyugo)
	&BurnDrvMSX_007tld,			// 007 - The Living Daylights (Euro)
	&BurnDrvsg1k_jb007a,		// 007 James Bond (Jpn, v2.6, OMV)
	&BurnDrvsg1k_jb007,			// 007 James Bond (Jpn, v2.7, OMV)
	&BurnDrvsg1k_jb007t,		// 007 James Bond (Tw)
	&BurnDrvMSX_10yard,			// 10-Yard Fight (Jpn)
	&BurnDrvMSX_10yarda,		// 10-Yard Fight (Jpn, Alt)
	&BurnDrvGtmro,				// 1000 Miglia: Great 1000 Miles Rally (94/05/10)
	&BurnDrvGtmrb,				// 1000 Miglia: Great 1000 Miles Rally (94/05/26)
	&BurnDrvGtmra,				// 1000 Miglia: Great 1000 Miles Rally (94/06/13)
	&BurnDrvGtmr,				// 1000 Miglia: Great 1000 Miles Rally (94/07/18)
	&BurnDrvmd_12in1,			// 12 in 1
	&BurnDrvmd_13mahjan,		// 13 Ma Jiang - 98 Mei Shao Nu Pian (Chi)
	&BurnDrvmd_16tongnk,		// 16 Ton (Jpn, Game no Kandume MegaCD Rip)
	&BurnDrvmd_16ton,			// 16 Ton (Jpn, SegaNet)
	&BurnDrvmd_16zhan,			// 16 Zhang Ma Jiang (Chi)
	&BurnDrvMSX_180,			// 180 Degrees
.....
}

其中的BurnDriver 结构体代表引擎的通用结构体
struct BurnDriver {
	char* szShortName;			// The filename of the zip file (without extension)
	char* szParent;				// The filename of the parent (without extension, NULL if not applicable)
	char* szBoardROM;			// The filename of the board ROMs (without extension, NULL if not applicable)
	char* szSampleName;			// The filename of the samples zip file (without extension, NULL if not applicable)
	char* szDate;

	// szFullNameA, szCommentA, szManufacturerA and szSystemA should always contain valid info
	// szFullNameW, szCommentW, szManufacturerW and szSystemW should be used only if characters or scripts are needed that ASCII can't handle
	char*    szFullNameA; char*    szCommentA; char*    szManufacturerA; char*    szSystemA;
	wchar_t* szFullNameW; wchar_t* szCommentW; wchar_t* szManufacturerW; wchar_t* szSystemW;

	INT32 Flags;			// See burn.h
	INT32 Players;		// Max number of players a game supports (so we can remove single player games from netplay)
	INT32 Hardware;		// Which type of hardware the game runs on
	INT32 Genre;
	INT32 Family;
	INT32 (*GetZipName)(char** pszName, UINT32 i);				// Function to get possible zip names
	INT32 (*GetRomInfo)(struct BurnRomInfo* pri, UINT32 i);		// Function to get the length and crc of each rom
	INT32 (*GetRomName)(char** pszName, UINT32 i, INT32 nAka);	// Function to get the possible names for each rom
	INT32 (*GetSampleInfo)(struct BurnSampleInfo* pri, UINT32 i);		// Function to get the sample flags
	INT32 (*GetSampleName)(char** pszName, UINT32 i, INT32 nAka);	// Function to get the possible names for each sample
	INT32 (*GetInputInfo)(struct BurnInputInfo* pii, UINT32 i);	// Function to get the input info for the game
	INT32 (*GetDIPInfo)(struct BurnDIPInfo* pdi, UINT32 i);		// Function to get the input info for the game
	INT32 (*Init)(); INT32 (*Exit)(); INT32 (*Frame)(); INT32 (*Redraw)(); INT32 (*AreaScan)(INT32 nAction, INT32* pnMin);
	UINT8* pRecalcPal; UINT32 nPaletteEntries;										// Set to 1 if the palette needs to be fully re-calculated
	INT32 nWidth, nHeight; INT32 nXAspect, nYAspect;					// Screen width, height, x/y aspect
};
比如，当前引擎的名称，支持多少个玩家，游戏的宽高信息等，以及对应的一些函数指针，比如初始化，退出，界面的渲染，获取按键的信息获取rom的信息等 ,这样不同的引擎就要实现对应的函数指针的实现，

这里我们看一个BurnDrvgames88 
struct BurnDriver BurnDrvgames88 = {
	"88games", NULL, NULL, NULL, "1988",
	"'88 Games\0", NULL, "Konami", "GX861",
	NULL, NULL, NULL, NULL,
	BDF_GAME_WORKING, 4, HARDWARE_PREFIX_KONAMI, GBF_SPORTSMISC, 0,
	NULL, games88RomInfo, games88RomName, NULL, NULL, games88InputInfo, games88DIPInfo,
	DrvInit, DrvExit, DrvFrame, DrvDraw, DrvScan, &DrvRecalc, 0x800,
	304, 224, 4, 3
};
大致就是这个引擎的名称为88games,最多支持的玩家数量为4个，游戏的分辨率，已经对应的函数指针,所以BurnLibInit 列出了当前支持的引擎数量

接着main函数继续执行 ,执行到 MediaInit(); 函数

int MediaInit()
{
	if (ScrnInit()) {					// Init the Scrn Window
		FBAPopupAddText(PUF_TEXT_DEFAULT, MAKEINTRESOURCE(IDS_ERR_UI_WINDOW));
		FBAPopupDisplay(PUF_TYPE_ERROR);
		return 1;
	}

	if (!bInputOkay) {
		InputInit();					// Init Input
	}

	nAppVirtualFps = nBurnFPS;

	if (!bAudOkay) {
		AudSoundInit();					// Init Sound (not critical if it fails)
	}

	nBurnSoundRate = 0;					// Assume no sound
	pBurnSoundOut = NULL;
	if (bAudOkay) {
		nBurnSoundRate = nAudSampleRate[nAudSelect];
		nBurnSoundLen = nAudSegLen;
	}

	if (!bVidOkay) {

		// Reinit the video plugin
		VidInit();
		if (!bVidOkay && nVidFullscreen) {

			nVidFullscreen = 0;

			MediaExit();
			return (MediaInit());
		}
		if (!nVidFullscreen) {
			ScrnSize();
		}

		if (!bVidOkay) {
			// Make sure the error will be visible
			SplashDestroy(1);

			FBAPopupAddText(PUF_TEXT_DEFAULT, MAKEINTRESOURCE(IDS_ERR_UI_VID_MODULE), VidGetModuleName());
			FBAPopupDisplay(PUF_TYPE_ERROR);
		}

		if (bVidOkay && ((bRunPause && bAltPause) || !bDrvOkay)) {
			VidRedraw();
		}
	}
	return 0;
}

首先初始化窗口，设置窗口对应的事件等
// Init the screen window (create it)
int ScrnInit()
{
	...
	if (ScrnRegister() != 0) {
		return 1;
	}
	....
	hScrnWnd = CreateWindowEx(nWindowExStyles, szClass, _T(APP_TITLE), nWindowStyles,
		0, 0, 0, 0,									   			// size of window
		NULL, NULL, hAppInst, NULL);
	....	
}
上面的代码主要是创建对应的显示窗口，并且设置对应的事件触发函数 ScrnRegister 函数，执行窗口的注册，包括事件等
static int ScrnRegister()
{
	WNDCLASSEX WndClassEx;
	ATOM Atom = 0;

	// Register the window class
	memset(&WndClassEx, 0, sizeof(WndClassEx)); 		// Init structure to all zeros
	WndClassEx.cbSize			= sizeof(WndClassEx);
	WndClassEx.style			= CS_HREDRAW | CS_VREDRAW | CS_DBLCLKS | CS_CLASSDC;// These cause flicker in the toolbar
	WndClassEx.lpfnWndProc		= ScrnProc;
	WndClassEx.hInstance		= hAppInst;
	WndClassEx.hIcon			= LoadIcon(hAppInst, MAKEINTRESOURCE(IDI_APP));
	WndClassEx.hCursor			= LoadCursor(NULL, IDC_ARROW);
	WndClassEx.hbrBackground	= static_cast<HBRUSH>( GetStockObject ( BLACK_BRUSH ));
	WndClassEx.lpszClassName	= szClass;

	// Register the window class with the above information:
	Atom = RegisterClassEx(&WndClassEx);
	if (Atom) {
		return 0;
	} else {
		return 1;
	}
}

窗口的注册事件主要是通过 RegisterClassEx 实现的，具体可以查对应的api,  lpfnWndProc：代表窗口处理函数的指针。,也即是对应的ScrnProc 函数
static LRESULT CALLBACK ScrnProc(HWND hWnd, UINT Msg, WPARAM wParam, LPARAM lParam)
{
	...
	switch (Msg) {
		HANDLE_MSG(hWnd, WM_CREATE,			OnCreate);
		//HANDLE_MSG(hWnd, WM_ACTIVATEAPP,	OnActivateApp);
		HANDLE_MSGB(hWnd,WM_PAINT,			OnPaint);
		HANDLE_MSG(hWnd, WM_CLOSE,			OnClose);
		HANDLE_MSG(hWnd, WM_DESTROY,		OnDestroy);
		HANDLE_MSG(hWnd, WM_COMMAND,		OnCommand);
	...
}
HANDLE_MSG是 Win32应用中的回调函数WndProc用于接收Windows向应用程序直接发送的消息，以及响应消息,这些事件对应的也即是窗口创建的时候，执行函数OnCreate,窗口绘制的时候，执行OnPaint，
窗口销毁的时候，执行OnDestroy 等，这里我们的窗口按钮的点击是在OnCommand 函数实现的，具体的窗口的绘制就不看了，都是利用window下特有的api实现的,这里主要看按钮的事件响应函数

static void OnCommand(HWND /*hDlg*/, int id, HWND /*hwndCtl*/, UINT codeNotify)
{
        case MENU_LOAD://普通的单机选择游戏事件
        {
			.....
			nGame = SelDialog(0, hScrnWnd);		// Bring up select dialog to pick a driver
			
			extern bool bDialogCancel;

			if (nGame >= 0 && bDialogCancel == false) {
				DrvExit();
				DrvInit(nGame, true);			// Init the game driver
				.....
				break;
			} 
			....
		}
		case MENU_STARTNET:  //网络对战的按钮事件
		{
			if (Init_Network()) {
				MessageBox(hScrnWnd, FBALoadStringEx(hAppInst, IDS_ERR_NO_NETPLAYDLL, true), FBALoadStringEx(hAppInst, IDS_ERR_ERROR, true), MB_OK);
				break;
			}

			if (!kNetGame) {
				InputSetCooperativeLevel(false, bAlwaysProcessKeyboardInput);
				AudBlankSound();
				SplashDestroy(1);
				StopReplay();
				DrvExit();
				DoNetGame();
				MenuEnableItems();
				InputSetCooperativeLevel(false, bAlwaysProcessKeyboardInput);
			}
			break;
		}
	.....	
	}
}

接着继续分析 MediaInit函数 InputInit() 也即是输入的初始化 
INT32 InputInit()
{
	INT32 nRet;

	bInputOkay = false;

	if (nInputSelect >= INPUT_LEN) {
		return 1;
	}

	if ((nRet = pInputInOut[nInputSelect]->Init()) == 0) {
		bInputOkay = true;
	}

	return nRet;
}

这里的pInputInOut 为
static struct InputInOut *pInputInOut[]=
{
#if defined (BUILD_WIN32)
	&InputInOutDInput,
#elif defined (BUILD_SDL)
	&InputInOutSDL,
#elif defined (_XBOX)
	&InputInOutXInput2,
#elif defined (BUILD_QT)
    &InputInOutQt,
#endif
};

可以看出这里提供了对应的平台下的实现，这里是win32 所以这个函数返回 InputInOutDInput，而这个结构体变量的定义为
struct InputInOut InputInOutDInput = { init, exit, setCooperativeLevel, newFrame, getState, readGamepadAxis, readMouseAxis, find, getControlName, NULL, _T("DirectInput8 input") }; 
所以执行到了对应的init，这是一个函数指针
int init()
{
	...
	if (FAILED(_DirectInput8Create(hAppInst, DIRECTINPUT_VERSION, IID_IDirectInput8, (void**)&pDI, NULL))) {
		return 1;
	}

	// keyboard
	if (FAILED(pDI->CreateDevice(GUID_SysKeyboard, &keyboardProperties[0].lpdid, NULL))) {
		return 1;
	}
	...
}

所以对于MediaInit函数 中的AudSoundInit 函数，这个函数主要完成声音的初始化，跟上面的执行逻辑是一样的
INT32 AudSoundInit()
{
	INT32 nRet;

	if (nAudSelect >= AUD_LEN) {
		return 1;
	}
	
	nAudActive = nAudSelect;

	if ((nRet = pAudOut[nAudActive]->SoundInit()) == 0) {
		bAudOkay = true;
	}

	return nRet;
}

MediaInit函数 中的VidInit(); 函数，这个主要完成视频的初始化
INT32 VidInit()
{
	....
	pVidOut[nVidActive]->Init() 
	...
}
static struct VidOut *pVidOut[] = {
#if defined (BUILD_WIN32)
	&VidOutDDraw,
	&VidOutD3D,
	&VidOutDDrawFX,
	&VidOutDX9,
	&VidOutDX9Alt,
#elif defined (BUILD_SDL)
	&VidOutSDLOpenGL,
	&VidOutSDLFX,
#elif defined (_XBOX)
	&VidOutD3D,
#elif defined (BUILD_QT)
    &VidOutOGL,
#endif
};
所以之类的初始化过程，跟上面也是类似的

接着回到主函数的 RunMessageLoop();					// Run the application message loop 
// The main message loop
int RunMessageLoop()
{
	int bRestartVideo;
	MSG Msg;

	do {
        ...
		//显示窗口
		ShowWindow(hScrnWnd, nAppShowCmd);									
		....
		//一直循环，用来响应各种事件
		while (1) {
		if (PeekMessage(&Msg, NULL, 0, 0, PM_REMOVE)) {
			// A message is waiting to be processed
			if (Msg.message == WM_QUIT)	{											// Quit program
				break;
			}
			if (Msg.message == (WM_APP + 0)) {										// Restart video
				bRestartVideo = 1;
				break;
			}
			.....
		}
		else {
			//没有事件，默认执行这里
			bRunPause ? wav_pause(false) : wav_pause(true);
			// No messages are waiting
			SplashDestroy(0);
			RunIdle();
		}
	}
}
PeekMessage是一个Windows API函数。该函数为一个消息检查线程消息队列，并将该消息(如果存在)放于指定的结构。 由于这里是刚初始化，所以没有事件，调用ShowWindow(hScrnWnd, nAppShowCmd); 
那么默认的窗口也就显示出来了,如果没有事件的化，默认执行 RunIdle ,RunIdle 会执行 RunFrame来更新界面

// With or without sound, run one frame.
// If bDraw is true, it's the last frame before we are up to date, and so we should draw the screen
static int RunFrame(int bDraw, int bPause)
{
	static int bPrevPause = 0;
	static int bPrevDraw = 0;

	if (bPrevDraw && !bPause) {
		VidPaint(0);							// paint the screen (no need to validate)
	}

	if (!bDrvOkay) {//由于我们还没有加载对应的引擎，所以会执行到这里
		return 1;
	}
	.....
}

上面就是显示默认的流程，现在点击加载游戏，查看对应的流程,也即是上面分析的onCommand中的 第一个case语句
static void OnCommand(HWND /*hDlg*/, int id, HWND /*hwndCtl*/, UINT codeNotify)
{
	if (bLoading) {
		return;
	}

	switch (id) {
	    case MENU_LOAD://普通的单机选择游戏事件
        {
			...
			nGame = SelDialog(0, hScrnWnd);		// Bring up select dialog to pick a driver
			
			extern bool bDialogCancel;

			if (nGame >= 0 && bDialogCancel == false) {
				DrvExit();
				DrvInit(nGame, true);			// Init the game driver
				.....
				break;
			} 
			...
        }
        ...
    }
	...
}

首先显示选择游戏的对话框 	nGame = SelDialog(0, hScrnWnd); 当游戏关闭的时候，会返回选择的游戏对应的索引
DrvInit(nGame, true);初始化对应的引擎

int DrvInit(int nDrvNum, bool bRestore)
{
	...
	DrvExit();						// 确保退出
	MediaExit();

	nBurnDrvActive = nDrvNum;		// Set the driver number	  保存当前下载的游戏引擎对应的下标索引
	
	...
	
	// Define nMaxPlayers early; GameInpInit() needs it (normally defined in DoLibInit()).
	nMaxPlayers = BurnDrvGetMaxPlayers();
	GameInpInit();					// Init game input
	...
 
	bDrvOkay = 1;						// Okay to use all BurnDrv functions   初始化完成之后，标识引擎准备好了
	...
}

BurnDrvGetMaxPlayers(); 获取到当前游戏引擎支持的游戏玩家数量,其实很简单，就从支持的引擎结构体数组中，获取到对应的引擎结构体，然后获取到对应的成员
extern "C" INT32 BurnDrvGetMaxPlayers()
{
	return pDriver[nBurnDrvActive]->Players;
}

GameInpInit();执行初始化游戏按键信息
INT32 GameInpInit()
{
	// Count the number of inputs  计算输入按键的数量
	nGameInpCount = 0;
	nMacroCount = 0;
	nMaxMacro = nMaxPlayers * 52;

	for (UINT32 i = 0; i < 0x1000; i++) {
		nRet = BurnDrvGetInputInfo(NULL,i);
		if (nRet) {														// end of input list
			nGameInpCount = i;
			break;
		}
	}

	// Allocate space for all the inputs  给当前游戏最多支持输入按键分配内存空间
	INT32 nSize = (nGameInpCount + nMaxMacro) * sizeof(struct GameInp);
	GameInp = (struct GameInp*)malloc(nSize);
	if (GameInp == NULL) {
		return 1;
	}
	memset(GameInp, 0, nSize);
	
	//给上面分配的内存按键赋值操作，赋值引擎按键的真正的内容
	GameInpBlank(1);
	...
}

nRet = BurnDrvGetInputInfo(NULL,i); 获取到按键的信息
extern "C" INT32 BurnDrvGetInputInfo(struct BurnInputInfo* pii, UINT32 i)	// Forward to drivers function
{
	return pDriver[nBurnDrvActive]->GetInputInfo(pii, i);
}
也即是对应的引擎结构体数组中，获取到对应的引擎结构体，然后获取到对应的成员，这里假设当前的引擎结构体为
struct BurnDriver BurnDrvgames88 = {
	"88games", NULL, NULL, NULL, "1988",
	"'88 Games\0", NULL, "Konami", "GX861",
	NULL, NULL, NULL, NULL,
	BDF_GAME_WORKING, 4, HARDWARE_PREFIX_KONAMI, GBF_SPORTSMISC, 0,
	NULL, games88RomInfo, games88RomName, NULL, NULL, games88InputInfo, games88DIPInfo,
	DrvInit, DrvExit, DrvFrame, DrvDraw, DrvScan, &DrvRecalc, 0x800,
	304, 224, 4, 3
};
那么这个成员对应的也即是games88InputInfo函数指针,这个函数指针的定义是在宏里面定义的 也即是 STDINPUTINFO(games88) 下面是这个宏的具体实现 宏里面##代表字符串的拼接
#define STDINPUTINFO(Name)												\
static INT32 Name##InputInfo(struct BurnInputInfo* pii, UINT32 i)		\
{																		\
	if (i >= sizeof(Name##InputList) / sizeof(Name##InputList[0])) {	\
		return 1;														\
	}																	\
	if (pii) {															\
		*pii = Name##InputList[i];										\
	}																	\
	return 0;															\
}
所以宏STDINPUTINFO(games88) 展开大致是这样的 
static INT32 games88InputInfo(struct BurnInputInfo* pii, UINT32 i)		
{																		
	if (i >= sizeof(games88InputList) / sizeof(games88InputList[0])) {	
		return 1;														
	}																	
	if (pii) {															
		*pii = games88InputList[i];										
	}																	
	return 0;															
}
而games88InputList 的定义为，其他的函数指针也是大致的逻辑
static struct BurnInputInfo games88InputList[] = {
	{"P1 Coin",		BIT_DIGITAL,	DrvJoy1 + 0,	"p1 coin"	},
	{"P1 Start",		BIT_DIGITAL,	DrvJoy2 + 3,	"p1 start"	},
	{"P1 Button 1",		BIT_DIGITAL,	DrvJoy2 + 0,	"p1 fire 1"	},
	{"P1 Button 2",		BIT_DIGITAL,	DrvJoy2 + 1,	"p1 fire 2"	},
	{"P1 Button 3",		BIT_DIGITAL,	DrvJoy2 + 2,	"p1 fire 3"	},

	{"P2 Coin",		BIT_DIGITAL,	DrvJoy1 + 1,	"p2 coin"	},
	{"P2 Start",		BIT_DIGITAL,	DrvJoy2 + 7,	"p2 start"	},
	{"P2 Button 1",		BIT_DIGITAL,	DrvJoy2 + 4,	"p2 fire 1"	},
	{"P2 Button 2",		BIT_DIGITAL,	DrvJoy2 + 5,	"p2 fire 2"	},
	{"P2 Button 3",		BIT_DIGITAL,	DrvJoy2 + 6,	"p2 fire 3"	},
	....
};

BurnInputInfo 结构体定义为 
struct BurnInputInfo {
	char* szName;
	UINT8 nType;
	union {
		UINT8* pVal;					// Most inputs use a char*
		UINT16* pShortVal;				// All analog inputs use a short*
	};
	char* szInfo;
};

DrvJoy1，DrvJoy2,DrvJoy3 本质为一个数组，所以对于上面结构体的第三个成员，类似的DrvJoy1 代表这个数组的首地址 +0或者1等代表指针的偏移，也即是对应的数组的成员
static UINT8 DrvJoy1[8];
static UINT8 DrvJoy2[8];
static UINT8 DrvJoy3[8];
static UINT8 DrvDips[3];
static UINT8 DrvInputs[3];

所以对于DrvJoy1 + 0 代表 DrvJoy1数组的第一个成员的地址  DrvJoy2 + 3代表这个数组的第四个成员的地址

继续分析 GameInpBlank(1); 函数的实现
INT32 GameInpBlank(INT32 bDipSwitch)
{
	UINT32 i = 0;
	struct GameInp* pgi = NULL;

	// Reset all inputs to undefined (even dip switches, if bDipSwitch==1)
	if (GameInp == NULL) {
		return 1;
	}
	
	//遍历上一步分配的按键的内存数组，完成赋值操作
	// Get the targets in the library for the Input Values
	for (i = 0, pgi = GameInp; i < nGameInpCount; i++, pgi++) {
		struct BurnInputInfo bii;
		memset(&bii, 0, sizeof(bii));
		BurnDrvGetInputInfo(&bii, i);
		if (bDipSwitch == 0 && (bii.nType & BIT_GROUP_CONSTANT)) {		// Don't blank the dip switches
			continue;
		}

		memset(pgi, 0, sizeof(*pgi));									// Clear input

		pgi->nType = bii.nType;											// store input type				存储按键的类型
		pgi->Input.pVal = bii.pVal;										// store input pointer to value  存储按键的指针，方便我们后面修改按键的内容

		if (bii.nType & BIT_GROUP_CONSTANT) {							// Further initialisation for constants/DIPs
			pgi->nInput = GIT_CONSTANT;
			pgi->Input.Constant.nConst = *bii.pVal;
		}
	}
	...
	return 0;
}

这里重点注意下 pgi->Input.pVal = bii.pVal;	 前面已经分析过了 bii.pVal 代表对应引擎对应的按键的地址，类似于DrvJoy1 + 0 代表 DrvJoy1数组的第一个成员的地址
那么当这个赋值操作完成之后，那么 pgi->Input.pVal 也指向了这个引擎的按键地址，后面当要修改按键的内容的时候，就可以直接修改pgi->Input.pVal中所代表的内容

继续回到 case语句块继续往下执行  nStatus = DoLibInit();	初始化对应的游戏引擎
static int DoLibInit()					// Do Init of Burn library driver
{
	int nRet = 0;

	//加载引擎对应的zip包
	if (DrvBzipOpen()) {
		return 1;
	}
	
	if ((BurnDrvGetHardwareCode() & HARDWARE_PUBLIC_MASK) != HARDWARE_SNK_MVS) {
		if (!bQuietLoading) ProgressCreate();
	}

	//执行对应的引擎初始化操作
	nRet = BurnDrvInit();

	//关闭引擎对应的zip包
	BzipClose();

	if (!bQuietLoading) ProgressDestroy();

	if (nRet) {
		return 3;
	} else {
		return 0;
	}
}

// Init game emulation (loading any needed roms)
extern "C" INT32 BurnDrvInit()
{
	...
	CheatInit();
	HiscoreInit();
	BurnStateInit();
	BurnInitMemoryManager();
	BurnRandomInit();

	//调用对应的引擎初始化，具体就不看了，逻辑就是获取到对应的引擎结构体，执行对应的函数指针
	nReturnValue = pDriver[nBurnDrvActive]->Init();	// Forward to drivers function

	nMaxPlayers = pDriver[nBurnDrvActive]->Players;
	....
}

引擎初始化成功之后，int DrvInit(int nDrvNum, bool bRestore)函数后面，会将 bDrvOkay = 1; 标识引擎准备好了，那么继续回调前面分析的RumLooperMessage

当再次执行到 RunIdle();的时候，也即是当前没有任何的按键操作的情况下，界面默认的显示的时候，会调用这个方法渲染游戏的界面,最终会调用到RunFrame 

static int RunFrame(int bDraw, int bPause)
{
	static int bPrevPause = 0;
	static int bPrevDraw = 0;

	if (bPrevDraw && !bPause) {
		VidPaint(0);							// paint the screen (no need to validate)
	}

	if (!bDrvOkay) {//判断是否引擎准备好了，此时已经准备好了
		return 1;
	}
	
	....
	GetInput(true);					// Update inputs  获取到键盘的按键内容
	
	... 
	BurnDrvFrame(); 				//渲染按键
	....
	
}

GetInput(true);	这里先分析下，按键的信息的获取
static int GetInput(bool bCopy)
{
	...
	InputMake(bCopy); 						// get input
	...
}
// This will process all PC-side inputs and optionally update the emulated game side.
INT32 InputMake(bool bCopy)
{
	...
	for (i = 0, pgi = GameInp; i < nGameInpCount; i++, pgi++) {
		if (pgi->Input.pVal == NULL) {
			continue;
		}

		switch (pgi->nInput) {
			....
			case GIT_SWITCH: {						// Digital input  默认的按键情况
				INT32 s = CinpState(pgi->Input.Switch.nCode);		  判断当前的按键 被按下的情况  ，具体会调用到window 中的 GetDeviceState 获取到对应的状态

				if (pgi->nType & BIT_GROUP_ANALOG) {
					// Set analog controls to full
					if (s) {
						pgi->Input.nVal = 0xFFFF;   				如果被按下，赋值为  0xFFFF;    前面已经分析过，这个值指向的就是引擎的按键内容，所以这个改变会导致引擎的按键发生改变
					} else {
						pgi->Input.nVal = 0x0001;					如果没有被按下 赋值为  0x0001;	
					}
					if (bCopy) {
						*(pgi->Input.pShortVal) = pgi->Input.nVal;
					}
				} else {
					// Binary controls
					if (s) {
						pgi->Input.nVal = 1;
					} else {
						pgi->Input.nVal = 0;
					}
					if (bCopy) {
						*(pgi->Input.pVal) = pgi->Input.nVal;
					}
				}

				break;
			}
			case GIT_JOYSLIDER:	{					// Joystick slider      游戏手柄
				INT32 nSlider = pgi->Input.Slider.nSliderValue;
				if (pgi->nType == BIT_ANALOG_REL) {
					nSlider -= 0x8000;
					nSlider >>= 4;
				}

				pgi->Input.nVal = (UINT16)nSlider;
				if (bCopy) {
					*(pgi->Input.pShortVal) = pgi->Input.nVal;
				}
				break;
			}
			case GIT_MOUSEAXIS:						// Mouse axis			鼠标
				pgi->Input.nVal = (UINT16)(CinpMouseAxis(pgi->Input.MouseAxis.nMouse, pgi->Input.MouseAxis.nAxis) * nAnalogSpeed);
				if (bCopy) {
					*(pgi->Input.pShortVal) = pgi->Input.nVal;
				}
				break;
			}
			.....
		}
	}
	....
	return 0;
}

BurnDrvFrame函数的实现，也即是会调用到对应引擎的函数指针，完成渲染
extern "C" INT32 BurnDrvFrame()
{
	CheatApply();									// Apply cheats (if any)
	HiscoreApply();
	return pDriver[nBurnDrvActive]->Frame();		// Forward to drivers function
}

由于每次渲染界面都要去获取到对应的按键情况，然后去修改引擎原本的按键内容，执行对应的引擎渲染，那么引擎就能事实的获取到这些按键的情况,这就是大致的流程，当然还有很多细节
```

### 对战代码实现

```C++
static void OnCommand(HWND /*hDlg*/, int id, HWND /*hwndCtl*/, UINT codeNotify)
{
	...
	case MENU_STARTNET:
	{
		if (Init_Network()) {//判断加载对战库成功，如果成功，显示对战库的对话框
			MessageBox(hScrnWnd, FBALoadStringEx(hAppInst, IDS_ERR_NO_NETPLAYDLL, true), FBALoadStringEx(hAppInst, IDS_ERR_ERROR, true), MB_OK);
			break;
		}
		...
		DoNetGame();
		...
	}
	...		
}

Init_Network() 会判断当前是否有对战库
int Init_Network(void)
{
//#if defined (_UNICODE)
//	Kaillera_HDLL = LoadLibrary(L"kailleraclient.dll");
//#else
	Kaillera_HDLL = LoadLibrary("kailleraclient.dll");
//#endif

	if (Kaillera_HDLL != NULL)
	{
		Kaillera_Get_Version = (int (WINAPI *)(char *version)) GetProcAddress(Kaillera_HDLL, "_kailleraGetVersion@4");
		Kaillera_Init = (int (WINAPI *)()) GetProcAddress(Kaillera_HDLL, "_kailleraInit@0");
		Kaillera_Shutdown = (int (WINAPI *)()) GetProcAddress(Kaillera_HDLL, "_kailleraShutdown@0");
		Kaillera_Set_Infos = (int (WINAPI *)(kailleraInfos *infos)) GetProcAddress(Kaillera_HDLL, "_kailleraSetInfos@4");
		Kaillera_Select_Server_Dialog = (int (WINAPI *)(HWND parent)) GetProcAddress(Kaillera_HDLL, "_kailleraSelectServerDialog@4");
		Kaillera_Modify_Play_Values = (int (WINAPI *)(void *values, int size)) GetProcAddress(Kaillera_HDLL, "_kailleraModifyPlayValues@8");
		Kaillera_Chat_Send = (int (WINAPI *)(char *text)) GetProcAddress(Kaillera_HDLL, "_kailleraChatSend@4");
		Kaillera_End_Game = (int (WINAPI *)()) GetProcAddress(Kaillera_HDLL, "_kailleraEndGame@0");

		if ((Kaillera_Get_Version != NULL) && (Kaillera_Init != NULL) && (Kaillera_Shutdown != NULL) && (Kaillera_Set_Infos != NULL) && (Kaillera_Select_Server_Dialog != NULL) && (Kaillera_Modify_Play_Values != NULL) && (Kaillera_Chat_Send != NULL) && (Kaillera_End_Game != NULL))
		{			
			//执行了Kaillera_Init()函数，进行库的初始化
			Kaillera_Init();
			Kaillera_Initialised = 1;
			return 0;
		}

		FreeLibrary(Kaillera_HDLL);
	} else {
	}

	Kaillera_Get_Version = Empty_Kaillera_Get_Version;
	Kaillera_Init = Empty_Kaillera_Init;
	Kaillera_Shutdown = Empty_Kaillera_Shutdown;
	Kaillera_Set_Infos = Empty_Kaillera_Set_Infos;
	Kaillera_Select_Server_Dialog = Empty_Kaillera_Select_Server_Dialog;
	Kaillera_Modify_Play_Values = Empty_Kaillera_Modify_Play_Values;
	Kaillera_Chat_Send = Empty_Kaillera_Chat_Send;
	Kaillera_End_Game = Empty_Kaillera_End_Game;

	Kaillera_Initialised = 0;
	return 1;
}

上面会加载这个对战库的dll，然后获取到对应的函数指针,对应的也即是对战库提供的头文件的对应的函数指针,获取到之后，存储到本地变量中，方便下次使用
DLLEXP kailleraGetVersion(char *version);
/*
    kailleraInit:
    Call this method when your program starts
*/
DLLEXP kailleraInit();
/*
    kailleraShutdown:
    Call this method when your program ends
*/
DLLEXP kailleraShutdown();
....

之后执行 DoNetGame 完成对战库的设置
static void DoNetGame()
{
	kailleraInfos ki;
	char tmpver[128];
	char* gameList;

	....获取到当前引擎支持的游戏列表
	gameList = CreateKailleraList();

	ki.appName = tmpver;
	ki.gameList = gameList;
	//设置游戏成功的回调
	ki.gameCallback = &gameCallback;
	//设置收到聊天的回调
	ki.chatReceivedCallback = &kChatCallback;
	//设置客户端掉线的回调
	ki.clientDroppedCallback = &kDropCallback;
	ki.moreInfosCallback = NULL;

	//调用Kaillera对战库的方法，传递参数
	Kaillera_Set_Infos(&ki);
	//kailleraSetInfos(&ki);

	//显示对战库的对话框
	Kaillera_Select_Server_Dialog(NULL);
	//kailleraSelectServerDialog(NULL);
	....
}

这样就进入到了对战库的对话框了，而且可以看到如果我们要写测试demo，我们的测试代码可以像上面这样写,我们先看看启动游戏的回调
static int WINAPI gameCallback(char* game, int player, int numplayers)
{
	bool bFound = false;
	HWND hActive;

	//根据对战库传递过来的引擎的名称，从我们的引擎列表中查找是否有对应的引擎
	for (nBurnDrvActive = 0; nBurnDrvActive < nBurnDrvCount; nBurnDrvActive++) {

		char* szDecoratedName = DecorateGameName(nBurnDrvActive);

		if (!strcmp(szDecoratedName, game)) {
			bFound = true;
			break;
		}
	}

	//如果找不到，就销毁Kailler这个对战库
	if (!bFound) {
		Kaillera_End_Game();
		return 1;
	}
	
	//标识当前是对战的模式
	kNetGame = 1;
	...								
	DrvInit(nBurnDrvActive, false);						// Init the game driver
	ScrnInit();
	AudSoundPlay();										// Restart sound
	VidInit();
	SetFocus(hScrnWnd);
	...
	RunMessageLoop();
	...
}
上面的逻辑跟单机实现是很相似的，不同的就是，这里查找对应的引擎 对应的索引，是根据对战库传递的游戏名称来获取到的

相应的收到的聊天的回调，直接在界面上显示既可
static void WINAPI kChatCallback(char* nick, char* text)
{
	TCHAR szTemp[128];
	_sntprintf(szTemp, 128, _T("%.32hs "), nick);
	VidSAddChatMsg(szTemp, 0xBFBFFF, ANSIToTCHAR(text, NULL, 0), 0x7F7FFF);
}

这个对战库只是用来传递当前用户的按键情况，由于是实时的，所以响应的代码也是在界面的渲染地方
static int RunFrame(int bDraw, int bPause)
{
	....
	if (kNetGame) {
		GetInput(true);						// Update inputs   获取到当前的按键情况
		if (KailleraGetInput()) {			// Synchronize input with Kaillera   同步按键信息
			return 0;
		}
	}

	//完成界面的渲染
	BurnDrvFrame();
	...
}

KailleraGetInput() 后面的文章继续介绍
```

