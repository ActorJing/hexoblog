---
title: Linux嵌入式3-输入设备
date: 2024-01-14 19:04:22
tags:
categories: Linux嵌入式学习
toc: true
---

# 1、输入类设备描述

设备文件路径：/dev/input/event

查看设备事件命令：cat /proc/bus/input/devices

![image-20240114204650594](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240114204650594.png)

# 2、linux内核描述

### 输入设备描述事件

查看事件描述符在"**input-event-codes.h**"文件中，已经在linux/input.h中包含

```c
struct input_event {
#if (__BITS_PER_LONG != 32 || !defined(__USE_TIME_BITS64)) && !defined(__KERNEL__)
	struct timeval time;
#define input_event_sec time.tv_sec
#define input_event_usec time.tv_usec
#else
	__kernel_ulong_t __sec;
#if defined(__sparc__) && defined(__arch64__)
	unsigned int __usec;
	unsigned int __pad;
#else
	__kernel_ulong_t __usec;
#endif
#define input_event_sec  __sec
#define input_event_usec __usec
#endif
	__u16 type;
	__u16 code;
	__s32 value;
};
/*
 timeval 为事件上报时间
 type 为事件类型
 code 为具体事件
 value 为事件的值
 例如键盘按键KEY0按下时，type表示触发按键，code表示KEY0，value表示按下还是松开
*/
```

**数据同步：**

同步事件***EV_SYN***用于实现同步操作、告知接收者本轮上报的数据已经完整，例如触摸屏幕一次操作需要上报x轴坐标，y轴坐标、触摸点信息等，此时就需要同步事件。同步事件的type类型如下：

```c
#define SYN_REPORT 0
#define SYN_CONFIG 1
#define SYN_MT_REPORT 2
#define SYN_DROPPED 3
#define SYN_MAX 0xf
#define SYN_CNT (SYN_MAX+1)
```

**所有的事件上报完成后都需要再上报一个同步事件，一般是SYN_REPORT，value为0。**

# 3、读取开发板上报事件

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
// /dev/input/event2
int main(int argc, char *argv[]){
    char gpio_path[100];
    char file_path[100];
    struct input_event in_ev = {0};
    struct pollfd pfd;
    char val;
    int fd;
    //效验传参   ./exe /dev/input/event2
    if (2 != argc)
    {
        fprintf(stderr,"usage:%s <gpio> <value>\n", argv[0]);
        exit(-1);
    }
    //打开事件
    if (0 > (fd = open(argv[1],O_RDONLY)))
    {
        perror("open export error");
        exit(-1);
    }
    //循环阻塞时读取上报事件
    for (;;)
    {
        /* code */
        if (sizeof(struct input_event) != read(fd, &in_ev, sizeof(struct input_event)))
        {
            /* code */
            perror("read error!");
            exit(-1);
        }
        printf("type: %d code: %d value: %d\n",in_ev.type, in_ev.code, in_ev.value);
        
    }
    exit(0);
}
```

# 4、触摸屏上报事件分析

![image-20240114212415740](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240114212415740.png)

按下触摸屏后触发绝对位移事件EV_ABS（type=3）中的ABS_MT_TRACKING_ID（code=57）事件，value为78，表示有一个新的触点被创建，value为-1时表示触点松开，触点的ID为78，53和54分别表示x坐标和y坐标。

获取触摸屏信息  需要使用一个开放函数：ioctl（input/output control）

```c
int ioctl(int fd, unsigned long request, ...);
/*
 fd 是文件描述符，表示要控制的设备。
 request 是控制命令，通常是一个宏，定义了要执行的特定操作。
 可选的参数 ... 取决于特定的 ioctl 命令，可能包含输入参数、输出参数或者不需要参数。
*/
//查询触摸屏触点信息，存放在info中
struct input_absinfo info;
if(0 > ioctl(fd, EVIOCGABS(ABS_MT_SLOT), &info)){
        perror("ioctl error");
        exit(-1);
    }
/* 常用来处理陀螺仪数据
struct input_absinfo {
	__s32 value;
	__s32 minimum;
	__s32 maximum;
	__s32 fuzz;
	__s32 flat;
	__s32 resolution;
};
*/
```

获取触摸点程序源码：

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
// /dev/input/event2
int main(int argc, char *argv[]){
    char gpio_path[100];
    char file_path[100];
    struct input_absinfo info;
    struct pollfd pfd;
    int max_slots;
    int fd;
    if (2 != argc)
    {
        fprintf(stderr,"usage:%s <gpio> <value>\n", argv[0]);
        exit(-1);
    }
    if (0 > (fd = open(argv[1],O_RDONLY)))
    {
        perror("open export error");
        exit(-1);
    }
    if(0 > ioctl(fd, EVIOCGABS(ABS_MT_SLOT), &info)){
        perror("ioctl error");
        exit(-1);
    }
    max_slots = info.maximum - info.minimum;
    printf("max slots: %d\n",max_slots);
    exit(0);
}
```

# 5、单点触摸测试

参考多点触摸实验

# 6、多点触摸测试

多点实验中，触摸点信息上报流程：

![image-20240118215126784](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240118215126784.png)

```
第一个触摸点直接上报ABS_MT_TRACKING_ID，ID只需知道是否为0，-1，大于0即可，具体编号不需要关心
出现第二个触摸点时，先上报ABS_MT_SLOT槽位信息，再上报坐标信息，最后上报ABS_MT_TRACKING_ID
如果另一个触摸点发生移动时，先上传ABS_MT_SLOT，再上传坐标信息，相同触摸点移动不上报ABS_MT_SLOT
```

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
// 多点触摸实验
// /dev/input/event1
//每个触摸点信息，valid为1时表示触摸点状态更新
struct ts_mt
{
    int x;
    int y;
    int id; // 表示触摸屏的唯一ID ABS_MT_TRACKING_ID
    int valid;
};
//缓存记录坐标，等待上报同步事件后再将坐标写在ts_mt结构体数组返回
struct tp_xy
{
    int x;
    int y;
};
/* 读取一次同步事件的触摸点坐标
fd: 时间句柄
max_slots： 最大触摸点数
mt: ts_mt数组指针
*/
static int ts_read(const int fd, const int max_slots, struct ts_mt *mt)
{
    //上报事件
    struct input_event in_ev;
    static int slot = 0;
    static struct tp_xy xy[12] = {0};
    int i;
    //清空ts_mt数组指针内容
    memset(mt, 0x0, max_slots*sizeof(struct ts_mt));
    // 设置id为-2，不为0表示触摸点按下，-1表示触摸点松开
    for (i = 0; i < max_slots; i++)
        mt[i].id = -2;

    for (;;)
    {
        if(sizeof(struct input_event) != read(fd, &in_ev, sizeof(struct input_event))){
            perror("read error");
            return -1;
        }
        switch (in_ev.type)
        {
            // 判断绝对位移事件
            case EV_ABS:
                switch (in_ev.code)
                {
                        // 触摸点槽位 code=47
                    case ABS_MT_SLOT:
                        slot = in_ev.value;
                        break;
                        // X轴坐标 code=53
                    case ABS_MT_POSITION_X:
                        xy[slot].x = in_ev.value;
                        mt[slot].valid = 1;
                        break;
                        // y轴坐标 code=54
                    case ABS_MT_POSITION_Y:
                        xy[slot].y = in_ev.value;
                        mt[slot].valid = 1;
                        break;
                        // 触摸点ID code=57 等于0表示移动事件
                    case ABS_MT_TRACKING_ID:
                        mt[slot].id = in_ev.value;
                        mt[slot].valid = 1;
                        break;
                }
                break;
            // 判断同步事件
            case EV_SYN:
                if(SYN_REPORT==in_ev.code){
                    //将记录的坐标更新到mt中
                    for (size_t i = 0; i < max_slots; i++)
                    {
                        mt[i].x = xy[i].x;
                        mt[i].y = xy[i].y;
                    }
                }
                return 0;
        }
    }
}


int main(int argc, char *argv[]){
    char gpio_path[100];
    char file_path[100];
    struct input_absinfo info;
    struct ts_mt *mt = NULL;
    struct pollfd pfd;
    int max_slots;
    int fd;
    if (2 != argc)
    {
        fprintf(stderr,"usage:%s <gpio> <value>\n", argv[0]);
        exit(-1);
    }

    if (0 > (fd = open(argv[1],O_RDONLY)))
    {
        perror("open export error");
        exit(-1);
    }

    if(0 > ioctl(fd, EVIOCGABS(ABS_MT_SLOT), &info)){
        perror("ioctl error");
        exit(-1);
    }
    max_slots = info.maximum + 1 - info.minimum;
    printf("max slots: %d\n",max_slots);
	//创建长度为max_slots的触摸点信息数组
    mt = calloc(max_slots, sizeof(struct ts_mt));
    for (; ; )
    {
        if(0>ts_read(fd, max_slots, mt))
            break;
        for (size_t i = 0; i < max_slots; i++)
        {
            //判断第i个触摸点的状态是否发生改变
            if (mt[i].valid)
            {
                if (0<=mt[i].id)
                    printf("slot<%d>, 按下(%d, %d)\n", i, mt[i].x, mt[i].y);
                else if(-1==mt[i].id)
                    printf("slot<%d>, 松开\n", i);
                else
                    printf("slot<%d>, 移动(%d, %d)\n", i, mt[i].x, mt[i].y);
                
            }
            
        }
        
    }
    close(fd);
    free(mt);
    exit(EXIT_FAILURE);
    
    exit(0);
}
```

