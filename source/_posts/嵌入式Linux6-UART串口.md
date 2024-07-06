---
title: 嵌入式Linux6-UART串口
date: 2024-02-06 23:09:33
tags:
categories: Linux嵌入式学习
toc: true
---

# UART串口

---

## 1、UART串口配置结构体

```c
struct termios
  {
    tcflag_t c_iflag;		/* input mode flags */
    tcflag_t c_oflag;		/* output mode flags */
    tcflag_t c_cflag;		/* control mode flags */
    tcflag_t c_lflag;		/* local mode flags */
    cc_t c_line;			/* line discipline */
    cc_t c_cc[NCCS];		/* control characters */
    speed_t c_ispeed;		/* input speed */
    speed_t c_ospeed;		/* output speed */
#define _HAVE_STRUCT_TERMIOS_C_ISPEED 1
#define _HAVE_STRUCT_TERMIOS_C_OSPEED 1
  };
```

串口属于一种终端设备，除此之外还包括常见的ssh等，它们都遵循终端统一的结构体termios，
