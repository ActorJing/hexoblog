---
title: Linux嵌入式4-tslib库函数
date: 2024-01-18 21:53:01
tags:
categories: Linux嵌入式学习
toc: true
---

# 1、tslib简介

![image-20240118215432029](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240118215432029.png)

tslib是为触摸屏设备开发的linux应用层库函数，通过配置conf文件适配触摸屏信息，通过调用函数得到触摸屏的实时触摸点信息。tslib移植过程参考应用开发编程指南第18章。

# 2、tslib库函数介绍

配置、打开触摸屏设备函数：

```c
#include "tslib.h"
/*
dev_name: 设备节点
nonblock： 0为阻塞方法打开触摸屏设备，非0表示非阻塞
*/
struct tsdev *ts_open(const char *dev_name, int nonblock);
/*
参数与上面相同，区别是，dev_name可以设置为NULL，函数会在配置文件读取设备节点
*/
struct tsdev *ts_setup(const char *dev_name, int nonblock)
//关闭触摸屏设备
int ts_close(struct tsdev *);
//配置触摸屏设备
int ts_config(struct tsdev *ts)
//获取触摸屏事件句柄    
ts_fd(ts)
```

---

读取触摸屏数据函数：

```c
/*

*/
int ts_read(struct tsdev *ts, struct ts_sample *samp, int nr)
/*

*/
int ts_read_mt(struct tsdev *ts, struct ts_sample_mt **samp, int max_slots, int nr)
```

# 3、tslib多点触摸测试：

tslib流程：

1、配置触摸屏

```c
ts = ts_setup(NULL,0);
```

2、读取触摸屏信息，计算最大触摸点数，申请触摸点信息数组

```c
if(0 > ioctl(ts_fd(ts), EVIOCGABS(ABS_MT_SLOT), &info)){
        perror("ioctl error");
        exit(-1);
    }
	//获取最大触摸点
    max_slots = info.maximum + 1 - info.minimum;
    printf("max slots: %d\n",max_slots);
	//申请存储触摸点数组空间
    samp = calloc(max_slots, sizeof(struct ts_sample_mt));
```

3、读取触摸屏数据

```c
//读触摸屏数据
if(0>ts_read_mt(ts, &samp, max_slots, 1)){
    perror("ts_read error");
    ts_close(ts);
    exit(-1);
}
```

4、解算触摸屏坐标

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <poll.h>
#include <linux/input.h>
#include <tslib.h>
// 多点触摸实验
// /dev/input/event1


int main(int argc, char *argv[]){
    //tsdev是设备文件 tslib
    struct tsdev *ts = NULL;
    //sample是具体坐标信息
    struct ts_sample_mt *samp = NULL;
    struct input_absinfo info;
    struct ts_mt *mt = NULL;
    int max_slots;
    
    int pressure[12] = {0};
    //配置触摸屏
    ts = ts_setup(NULL,0);
    if(NULL==ts){
        perror("ts_setup error");
        exit(-1);
    }
	//ts_fd获取触摸屏事件句柄，再获取触摸屏信息
    if(0 > ioctl(ts_fd(ts), EVIOCGABS(ABS_MT_SLOT), &info)){
        perror("ioctl error");
        exit(-1);
    }
	//获取最大触摸点
    max_slots = info.maximum + 1 - info.minimum;
    printf("max slots: %d\n",max_slots);
	//申请存储触摸点数组空间
    samp = calloc(max_slots, sizeof(struct ts_sample_mt));
    for (;;)
    {
        //读触摸屏数据
        if(0>ts_read_mt(ts, &samp, max_slots, 1)){
            perror("ts_read error");
            ts_close(ts);
            exit(-1);
        }
        for (size_t i = 0; i < max_slots; i++)
        {
            //触摸点状态发生改变
            if(samp[i].valid)
            {
                //判断压力是否大于0 是表示按下 否表示松开
                if (samp[i].pressure)
                {
                    //上一次压力为0表示刚按下 否则表示移动
                    if(pressure[samp[i].slot]==0){
                        printf("%d按下：x=%d y=%d\n", samp[i].slot, samp[i].x, samp[i].y);
                    }
                    else{
                        printf("%d移动：x=%d y=%d\n", samp[i].slot, samp[i].x, samp[i].y);
                    }
                }
                else{
                    printf("%d松开\n", samp[i].slot);
                }
            }
            //更新历史压力
            pressure[samp[i].slot] = samp[i].pressure;
        }
        
        
    }
    
    ts_close(ts);
    free(samp);
    exit(0);
}
```

