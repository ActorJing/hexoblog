---
title: Linux嵌入式5-FrameBuffer应用编程
date: 2024-01-31 11:49:08
tags:
categories: Linux嵌入式学习
toc: true
---

# FrameBuffer应用编程

## 1、内存映射

用户程序操作文件的一般方式为调用系统库函数（open，read，write），库函数的流程为拷贝用户数据空间，到内核空间，获取文件句柄，返回用户空间，操作文件时，使用句柄到内核空间找到文件进行修改，每一次操作都是如此。用户空间和内核空间的机制导致数据传输速度不能最大，于是采用**内存映射**的方式。

用户程序可以直接访问内存，内存映射是在内存中申请空间对应物理文件，修改内存的数据会自动同步到物理文件，注意这个同步不是及时的，仍由内核调用，使用open这种库函数也不是及时同步，都是由内核调用同步，内存映射返回的是一个指针，可以直接访问和修改内容。

## 2、LCD应用编程流程

- 打开/dev/fdX设备文件
- 使用ioctl函数读取LCD参数信息
- 使用存储映射的方式将屏幕显示缓冲区映射到用户空间
- 直接读写显示缓冲区进行绘图和显示
- 完成显示后关闭存储映射，关闭设备文件

## 3、属性结构体介绍

```c
//显示设备的可变参数参数-一般表示显示屏显示大小，不变参数一般指硬件属性，比如缓冲区宽度
struct fb_var_screeninfo {
	__u32 xres;			/* visible resolution		*/
	__u32 yres;
	__u32 xres_virtual;		/* virtual resolution		*/
	__u32 yres_virtual;
	__u32 xoffset;			/* offset from virtual to visible */
	__u32 yoffset;			/* resolution			*/

	__u32 bits_per_pixel;		/* guess what			*/
	__u32 grayscale;		/* 0 = color, 1 = grayscale,	*/
					/* >1 = FOURCC			*/
	struct fb_bitfield red;		/* bitfield in fb mem if true color, */
	struct fb_bitfield green;	/* else only length is significant */
	struct fb_bitfield blue;
	struct fb_bitfield transp;	/* transparency			*/	

	__u32 nonstd;			/* != 0 Non standard pixel format */

	__u32 activate;			/* see FB_ACTIVATE_*		*/

	__u32 height;			/* height of picture in mm    */
	__u32 width;			/* width of picture in mm     */

	__u32 accel_flags;		/* (OBSOLETE) see fb_info.flags */

	/* Timing: All values in pixclocks, except pixclock (of course) */
	__u32 pixclock;			/* pixel clock in ps (pico seconds) */
	__u32 left_margin;		/* time from sync to picture	*/
	__u32 right_margin;		/* time from picture to sync	*/
	__u32 upper_margin;		/* time from sync to picture	*/
	__u32 lower_margin;
	__u32 hsync_len;		/* length of horizontal sync	*/
	__u32 vsync_len;		/* length of vertical sync	*/
	__u32 sync;			/* see FB_SYNC_*		*/
	__u32 vmode;			/* see FB_VMODE_*		*/
	__u32 rotate;			/* angle we rotate counter clockwise */
	__u32 colorspace;		/* colorspace for FOURCC-based modes */
	__u32 reserved[4];		/* Reserved for future compatibility */
};
//显示设备的固定参数
struct fb_fix_screeninfo {
	char id[16];			/* identification string eg "TT Builtin" */
	unsigned long smem_start;	/* Start of frame buffer mem */
					/* (physical address) */
	__u32 smem_len;			/* Length of frame buffer mem */
	__u32 type;			/* see FB_TYPE_*		*/
	__u32 type_aux;			/* Interleave for interleaved Planes */
	__u32 visual;			/* see FB_VISUAL_*		*/ 
	__u16 xpanstep;			/* zero if no hardware panning  */
	__u16 ypanstep;			/* zero if no hardware panning  */
	__u16 ywrapstep;		/* zero if no hardware ywrap    */
	__u32 line_length;		/* length of a line in bytes    */
	unsigned long mmio_start;	/* Start of Memory Mapped I/O   */
					/* (physical address) */
	__u32 mmio_len;			/* Length of Memory Mapped I/O  */
	__u32 accel;			/* Indicate to driver which	*/
					/*  specific chip/card we have	*/
	__u16 capabilities;		/* see FB_CAP_*			*/
	__u16 reserved[2];		/* Reserved for future compatibility */
};
//读取参数结构体
ioctl(fd, FBIOGET_VSCREENINFO, &fb_var);
ioctl(fd, FBIOGET_FSCREENINFO, &fb_fix);
```

## 4、存储映射

```c
#include <sys/mman.h>
void *mmap (void *__addr, size_t __len, int __prot,
		   int __flags, int __fd, __off_t __offset) __THROW;)
//地址自动分配，传入NULL，在存储映射中遇到一个问题，显示屏的line_length和显示宽度width不一样，显示屏line_length由硬件缓冲区大小决定，申请内存时，空间大小应该为line_length*height，而不是width*height
screen_base = mmap(NULL, screen_size, PROT_WRITE, MAP_SHARED, fd, 0);
//关闭存储映射
munmap(screen_base, screen_size);
```

## 5、显示效果

```c
//颜色标准转换
#define argb8888_to_rgb565(color) ({ \
    unsigned int temp = (color);\
    ((temp & 0xF80000UL) >> 8) | \
    ((temp & 0xFC00UL) >> 5) | \
    ((temp & 0xF8UL) >> 3); \
})
```

![8118a23003076a7df800a48e2338f7d](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/8118a23003076a7df800a48e2338f7d.jpg)

## 6、程序源码

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
#include <sys/mman.h>
#include <linux/fb.h>

#define argb8888_to_rgb565(color) ({ \
    unsigned int temp = (color);\
    ((temp & 0xF80000UL) >> 8) | \
    ((temp & 0xFC00UL) >> 5) | \
    ((temp & 0xF8UL) >> 3); \
})

static int width;
static int height;
static unsigned short *screen_base = NULL;
// 打点
static void lcd_draw_point(unsigned int x, unsigned int y, unsigned int color){
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    if(x>=width)
        x = width-1;
    if(y>=height)
        y = height-1;
    screen_base[y*width +x] = rgb565_color;
}
// 画线
static void lcd_draw_line(unsigned int x, unsigned int y, int dir, unsigned int length, unsigned int color){
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    unsigned int end;
    unsigned long temp;
    if(x>=width)
        x = width-1;
    if(y>=height)
        y = height-1;

    temp = y*width +x;

    if (dir){
        end = x + length -1;
        if(end>=width)
            end = width-1;
        for(; x<=end; x++, temp++)
            screen_base[temp] = rgb565_color;
    }
    else{
        end = y + length -1;
        if(end>=height)
            end = height-1;
        for(; y<end; y++, temp+=width)
            screen_base[temp] = rgb565_color;
    }
    
}
// 画矩形
static void lcd_draw_rectangle(unsigned int start_x, unsigned int end_x, unsigned int start_y, unsigned int end_y, unsigned int color){
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    int x_len = end_x - start_x + 1;
    int y_len = end_y - start_y - 1;

    lcd_draw_line(start_x, start_y, 1, x_len, color);
    lcd_draw_line(start_x, end_y, 1, x_len, color);
    lcd_draw_line(start_x, start_y+1, 0, y_len, color);
    lcd_draw_line(end_x, start_y+1, 0, y_len, color);
}
//区域填充
static void lcd_fill(unsigned int start_x, unsigned int end_x, unsigned int start_y, unsigned int end_y, unsigned int color){
    unsigned short rgb565_color = argb8888_to_rgb565(color);
    int x_len = end_x - start_x + 1;
    for(; start_y<=end_y; start_y++)
        lcd_draw_line(start_x, start_y, 1, x_len, color);
}

int main(int argc, char *argv[]){

    struct fb_fix_screeninfo fix_info;
    struct fb_var_screeninfo var_info;
    unsigned int screen_size;
    int fd;

    if (2 != argc)
    {
        fprintf(stderr,"usage:%s <event>\n", argv[0]);
        exit(-1);
    }

    if (0 > (fd = open(argv[1],O_RDWR)))
    {
        perror("open error");
        exit(-1);
    }

    if(0 > ioctl(fd, FBIOGET_VSCREENINFO, &var_info)){
        perror("ioctl error");
        exit(-1);
    } 
    if(0 > ioctl(fd, FBIOGET_FSCREENINFO, &fix_info)){
        perror("ioctl error");
        exit(-1);
    }
    screen_size = fix_info.line_length * var_info.yres;
    width = var_info.xres;
    height = var_info.yres;
    printf("frame config x:%d, y:%d\n", var_info.xres, var_info.yres);
    printf("frame config size:%d\n", fix_info.line_length);

    screen_base = mmap(NULL, screen_size, PROT_WRITE, MAP_SHARED, fd, 0);

    if(MAP_FAILED == (void *)screen_base){
        perror("mmap error");
        close(fd);
        exit(EXIT_FAILURE);
    }
    /* 画正方形方块 */
    int w = height * 0.25;//方块的宽度为 1/4 屏幕高度
    lcd_fill(0, width-1, 0, height-1, 0x0); //清屏（屏幕显示黑色）
    lcd_fill(0, w, 0, w, 0xFF0000); //红色方块
    lcd_fill(width-w, width-1, 0, w, 0xFF00); //绿色方块
    lcd_fill(0, w, height-w, height-1, 0xFF); //蓝色方块
    lcd_fill(width-w, width-1, height-w, height-1, 0xFFFF00);//黄色方块
    /* 画线: 十字交叉线 */
    lcd_draw_line(0, height * 0.5, 1, width, 0xFFFFFF);//白色线
    lcd_draw_line(width * 0.5, 0, 0, height, 0xFFFFFF);//白色线
    /* 画矩形 */
    unsigned int s_x, s_y, e_x, e_y;
    s_x = 0.25 * width;
    s_y = w;
    e_x = width - s_x;
    e_y = height - s_y;
    for ( ; (s_x <= e_x) && (s_y <= e_y);
    s_x+=5, s_y+=5, e_x-=5, e_y-=5)
    lcd_draw_rectangle(s_x, e_x, s_y, e_y, 0xFFFFFF);

    printf("frame draw over1\n");
    munmap(screen_base, screen_size);
    close(fd);

    exit(0);
}
```

