---
title: Linux嵌入式2-GPIO编程
date: 2024-01-14 12:42:33
tags:
categories: Linux嵌入式学习
toc: true
---

### 头文件说明

```c
<sys/types.h>：该头文件定义了一些基本的系统数据类型，如size_t、time_t等。
<sys/stat.h>：该头文件定义了一些关于文件状态的函数和宏，如stat()、S_IRUSR等。
<fcntl.h>：该头文件定义了一些文件控制操作的函数和常量，如open()、O_RDONLY等。
<unistd.h>：该头文件定义了一些系统调用函数，如read()、write()等。
```



### 函数说明

```c
int access(const char *pathname, int mode);
/*
mode 是要检查的权限模式。常用的权限模式有以下几种：

F_OK：用于检查文件或目录是否存在。
R_OK：用于检查文件或目录是否可读。
W_OK：用于检查文件或目录是否可写。
X_OK：用于检查文件或目录是否可执行。
存在返回0，不存在返回-1
*/   
    
len = strlen(argv[1]);
        if(len != write(fd,argv[1],len)){

        }
/*
write函数向文件写入数据，写入argv[1]字符串，后面接写入的字节数，成功返回写入的字节数
*/
```

## 一、GPIO应用编程

### 1、基础属性：

gpio设备目录在/sys/class/gpio中：

![image-20231015142048558](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20231015142048558.png)

gpiochip0~128分别对应i.max6ull的五组gpio1~5，export用来导出指定编号的gpio，加入需要导出GPIO4_IO20，首先需要确定GPIO的编号，GPIO4对应gpiochip96，编号为94+20=114

执行导出命令为echo 94 > export

以导出GPIO0_IO1为例：执行echo 1 > export，会生成一个gpio1的文件夹，里面描述了gpio1的相关信息：

![image-20231015142724436](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20231015142724436.png)

```c
dirction: IO的方向可以设置为out 和 in
active_low: 电平逻辑状态，默认为0，此时1为高电平 0为低电平
value： 电平状态
edge: 中断触发：none rising falling both分别表示无触发、上升沿、下降沿、边沿触发
```

### 2、gpio_config函数

```c
/*
	path：gpioX路径	/sys/class/gpio/gpio1
	attr：需要修改的属性 direction
	value：修改的具体值	out
*/
int gpio_config(const char *path, const char *attr, const char *val){
    int fd;
    int len;
    char file_path[100];
    // 拼接字符串
    sprintf(file_path,"%s/%s", path, attr);
    // 打开文件
    if (0 > (fd = open(file_path, O_WRONLY)))
    {
        perror("open gpio error");
        close(fd);
        return -1;
    }
    // 向文件写入指定长度的数据
    len = strlen(val);
    if (len != write(fd, val, len))
    {
        perror("write info error");
        close(fd);
        return -1;
    }
    close(fd);
    fprintf(stderr,"write success!\n");
    return 0;
}
// main函数：
// 判断输入的gpioX是否存在，不存在需要通过写export文件导出IO
sprintf(gpio_path,"/sys/class/gpio/gpio%s",argv[1]);
if (access(gpio_path,F_OK))
{
    int fd;
    int len;
    if (0 > (fd = open("/sys/class/gpio/export",O_WRONLY)))
    {
        perror("open export error");
        exit(-1);
    }
    len = strlen(argv[1]);
    if(len != write(fd,argv[1],len)){
        perror("write error");
        close(fd);
        exit(-1);
    }
    close(fd);
}
if(gpio_config(gpio_path,"direction", "in")){
    fprintf(stderr,"write direction error\n");
    exit(-1);
}
if(gpio_config(gpio_path,"active_low", "0")){
    fprintf(stderr,"write active_low error\n");
    exit(-1);
}
if(gpio_config(gpio_path,"value", argv[2])){
    fprintf(stderr,"write value error\n");
    exit(-1);
}
```

### 3、poll()函数

```c
// 使用poll()函数实现非阻塞式中断触发
int main(int argc, char *argv[]){
    char gpio_path[100];
    char file_path[100];
    // 创建pollfd结构体，描述文件就绪状态
    struct pollfd pfd;
    char val;
    int ret;
    if (2 != argc)
    {
        fprintf(stderr,"usage:%s <gpio> <value>\n", argv[0]);
        exit(-1);
    }
    sprintf(gpio_path,"/sys/class/gpio/gpio%s",argv[1]);
    if (access(gpio_path,F_OK))
    {
        int fd;
        int len;
        if (0 > (fd = open("/sys/class/gpio/export",O_WRONLY)))
        {
            perror("open export error");
            exit(-1);
        }
        len = strlen(argv[1]);
        if(len != write(fd,argv[1],len)){
            perror("write error");
            close(fd);
            exit(-1);
        }
        close(fd);
    }

    if(gpio_config(gpio_path,"direction", "in")){
        fprintf(stderr,"write direction error\n");
        exit(-1);
    }
    if(gpio_config(gpio_path,"active_low", "0")){
        fprintf(stderr,"write active_low error\n");
        exit(-1);
    }
    // 配置为边沿触发
    if(gpio_config(gpio_path,"edge", "both")){
        fprintf(stderr,"write edge error\n");
        exit(-1);
    }
    // 打开IO状态value文件，文件描述符保存在pfd.fd
    sprintf(file_path,"%s/%s", gpio_path, "value");
    if (0 > (pfd.fd = open(file_path,O_RDONLY)))
    {
        perror("open pfd.fd error");
        exit(-1);
    }
    
    pfd.events = POLLPRI;// 只关心高优先级数据可读 中断 只有高优先级才会触发文件转换为就绪态
    read(pfd.fd, &val, 1);// 读取一次清除状态
    //轮询读取
    for (;;)
    {
        // 监听pfd文件状态，内部有一个文件描述符 超时时间-1
        ret = poll(&pfd, 1, -1);
        if (0 > ret) {
            perror("poll error");
            exit(-1);
        }
        else if(0 == ret){
            fprintf(stderr,"poll time out");
            exit(-1);
        }
		// 事件触发
        if (pfd.revents & POLLPRI)
        {
            // 因为之前读取过文件，需要移动指针到0
            if(0 > lseek(pfd.fd, 0, SEEK_SET)){
                perror("lseek error");
                exit(-1);
            }
			// 读取文件
            if (0 > read(pfd.fd, &val, 1))
            {
                perror("read pfd.fd error");
                exit(-1);
            }
            printf("get interupt <value=%c>\n", val);

        }
        
    }
    
    exit(0);
}
```

