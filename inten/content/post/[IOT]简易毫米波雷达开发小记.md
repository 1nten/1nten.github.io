---
title: "[IOT]简易毫米波雷达开发小记"
date: "2021-06-08"
tags: ["IOT"]
ShowToc: true
TocOpen: true
draft: false
---

# [IOT]简易毫米波雷达开发小记

## 开发环境

**软件**

Visual Studio 2019

EasyX_2018春分版

FlyMcu.exe

Windows串口调试工具.exe

**硬件**

电脑（Windows）

定制的毫米波雷达电路板

Micro USB数据线

## 线路连接

用Micro USB数据线连接电路板和电脑即可

<img src="https://www.kro1lsec.com:442/images/2021/06/10/20210610190914.jpg" alt="IMG_20210610_114359"  />

电路板背面黑黑的正方形是主要的传感器

![IMG_20210610_114338](https://www.kro1lsec.com:442/images/2021/06/10/20210610191155.jpg)

部分电脑会缺少[CP210x_USB.exe驱动](https://www.drvsky.com/driver/CP2102.htm),下载安装即可

安装成功后，设备管理器就能识别出接上的电路板

![驱动安装成功](https://www.kro1lsec.com:442/images/2021/06/10/20210610191008.png)

## 串口烧写

打开FlyMcu.exe按照下图配置开始编程，等待几分钟后完成。

![flybaincheng](https://www.kro1lsec.com:442/images/2021/06/10/20210610191026.jpg)

然后再打开Windows串口调试工具，将电路板背面的传感器朝上，将手机之类的物品放置到上方15cm处。下图中绿色的数字即为雷达检测到的信息。但是这种显示方式很不直观，接下来我们编写可视化窗口。

![2](https://www.kro1lsec.com:442/images/2021/06/10/20210610191022.jpg)

## 可视化窗口编写

先配置VS2019+EasyX的开发环境[VS2019安装EasyX详细过程](https://blog.csdn.net/weixin_46204734/article/details/109185309)

如果缺少SDK，按照提示下载安装即可

```c
#include "pch.h"
#include <Windows.h>
#include <memory.h>
#include <string.h>
#include <winsock.h>
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <windows.h>

#include<graphics.h> //如果您需要在vc及vs环境中使用graphics.h的功能 您可以下载EasyX图形库
#include <conio.h>
#define X0 10
#define Y0 860
#define BUFSIZE 512
#pragma comment(lib,"ws2_32.lib")


using namespace std;

int port_setting();

HANDLE hCom;  //将一个任务切换到指定的线程中去执行

int main(void)
{
	if (!(port_setting()))
	{
		printf("串口初始化异常");
	}
	DWORD wCount;  //定义一个32位的无符号整数，用到头文件#include <windows.h>
	BOOL bReadStat;
	//数据处理
	int x, y;

	int FLAG = 0;
	char data[4000] = {0};
	int data_size = 0;
	//初始化图形系统 头文件graphics.h   
    initgraph(2560, 1600, SHOWCONSOLE ); 
	//以下局部代码创建一个尺寸为 1920x1080 的绘图环境，同时显示控制台窗口SHOWCONSOLE：2560 x 1600
	while (1)
	{
		PurgeComm(hCom,PURGE_TXCLEAR | PURGE_RXCLEAR);//清空缓冲区
		char str[BUFSIZE] = { 0 };	//设置能接收的数据的位数 和类型,位数要等于一个数据包位数，否则数据包将混叠
		bReadStat = ReadFile(hCom, str, 5, &wCount, NULL);//readfile() 函数读取一个文件，并写入到输出缓冲。
		if (!bReadStat)
		{
			printf("读串口失败!");
			return FALSE;
		} 
		//cleardevice();
		setcolor(RGB(255, 255, 255));//白色：rgb(255,255,255)
		moveto(X0, Y0);//从30，860开始画
		lineto(X0, Y0 -800);
		moveto(X0, Y0);
		lineto(X0 + 1300, Y0);
		moveto(X0, Y0);
		setcolor(RGB(0, 140, 140));

		/*2. 常用颜色的RGB值

白色：rgb(255,255,255)

黑色：rgb(0,0,0)

红色：rgb(255,0,0)

绿色：rgb(0,255,0)

蓝色：rgb(0,0,255)

青色：rgb(0,255,255)

紫色：rgb(255,0,255)*/
		for (int i = 1; i <20; i++)
		{
			moveto(X0, Y0 - i * 40);
			lineto(X0 + 1300, Y0 - i * 40);
		}
		moveto(X0, Y0);

		for (int index = 0; 1; index++) 
		{

			ReadFile(hCom, str, 5, &wCount, NULL);
			if (FLAG == 2)
			{
				FLAG = 0;
				cleardevice();
				setcolor(RGB(255, 255, 255));
				moveto(X0, Y0);
				lineto(X0, Y0 - 800);
				moveto(X0, Y0);
				lineto(X0 + 1300, Y0);
				moveto(X0, Y0);
				setcolor(RGB(0, 140, 140));
				for (int i = 1; i < 20; i++)
				{
					moveto(X0, Y0-i*40);
					lineto(X0 + 1300, Y0 - i * 40);
				}
				moveto(X0, Y0);
			}
			if (strcmp(str, "HDIAO") == 0)
			{
				setcolor(RGB(0, 255, 255));
			}
			if (strcmp(str, "ENDBB") == 0)
			{
				FLAG++;
				break;
			}
			
			setcolor(RGB(255, 255, 255));
			sscanf_s(str, "%d", &y);
			x = index*2 + X0;
			y = Y0 - y /9;//5
			lineto(x, y);
		}
		printf("ENDEND\n");
		Sleep(20);

		WSACleanup();

	}
}

int port_setting()
{
	    hCom = CreateFile(TEXT("COM3"),				//COM口
		GENERIC_READ | GENERIC_WRITE,			 //允许读和写
		0,										 //独占方式
		NULL,
		OPEN_EXISTING,							//打开而不是创建
		0,										//同步方式
		NULL);
	if (hCom == (HANDLE)-1)
	{
		printf("打开COM失败!\n");
		return FALSE;
	}
	else
	{
		printf("COM打开成功！\n");
	}
	SetupComm(hCom, 20000, 20000);		//设置读写缓冲区大小
	COMMTIMEOUTS TimeOuts;
	//设定读超时
	TimeOuts.ReadIntervalTimeout = 6000;
	TimeOuts.ReadTotalTimeoutMultiplier = 8000;
	TimeOuts.ReadTotalTimeoutConstant = 150000;
	//设定写超时
	TimeOuts.WriteTotalTimeoutMultiplier = 500;
	TimeOuts.WriteTotalTimeoutConstant = 2000;
	SetCommTimeouts(hCom, &TimeOuts);					//设置超时
	DCB dcb;
	GetCommState(hCom, &dcb);
	dcb.BaudRate = 115200;					//波特率 
	dcb.ByteSize = 8;						//每个字节有8位
	dcb.Parity = NOPARITY;					//无奇偶校验位
	dcb.StopBits = ONE5STOPBITS;			//一个停止位
	dcb.fNull = 0;							//忽略0
	SetCommState(hCom, &dcb);
	return 1;
}
```

## 最终效果

放置金属，陶瓷，木板等不同材质的物品，显示的波形也不同

横坐标：雷达与待测物体的距离

纵坐标：信号强度

![最终效果](https://www.kro1lsec.com:442/images/2021/06/10/20210610191016.png)