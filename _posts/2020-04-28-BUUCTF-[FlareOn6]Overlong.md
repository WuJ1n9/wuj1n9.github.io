---
layout:     post
title:      "BUUCTF-[FlareOn6]Overlong-WP"
subtitle:   "简单 patch"
date:       2020-04-28 12:28:00
author:     "WuJ"
header-img: "img/img13.jpg"
header-mask: 0
tags:
    - BUUCTF
    - 逆向
catalog: true
---

# BUUCTF-[FlareOn6]Overlong-WP

##  查看 PE 信息

![image-20200428141724975](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge9gh727lgj30sm0e477x.jpg)

## IDA 分析

```C
int __stdcall start(int a1, int a2, int a3, int a4)
{
  int v4; // eax
  CHAR Text[128]; // [esp+0h] [ebp-84h]
  int v7; // [esp+80h] [ebp-4h]

  v4 = sub_401160(Text, &unk_402008, 0x1Cu);
  v7 = v4;
  Text[v4] = 0;
  MessageBoxA(0, Text, Caption, 0);
  return 0;
}
```

程序本身非常简单，就是调用 Win32 API 弹出一个对话框

![image-20200428141920655](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge9gj5ignaj30b208caa4.jpg)

查看 sub_401160 函数 

```C
unsigned int __cdecl sub_401160(char *a1, int a2, unsigned int a3)
{
  int v3; // ST08_4
  unsigned int i; // [esp+4h] [ebp-4h]

  for ( i = 0; i < a3; ++i )
  {
    a2 += sub_401000(a1, a2);
    v3 = *a1++;
    if ( !v3 )
      break;
  }
  return i;
}
```

继续跟进

```c
signed int __cdecl sub_401000(_BYTE *a1, unsigned __int8 *a2)
{
  signed int v3; // [esp+0h] [ebp-8h]
  int v4; // [esp+4h] [ebp-4h]

  if ( (signed int)*a2 >> 3 == 30 )
  {
    v4 = a2[3] & 0x3F | ((a2[2] & 0x3F) << 6) | ((a2[1] & 0x3F) << 12) | ((*a2 & 7) << 18);
    v3 = 4;
  }
  else if ( (signed int)*a2 >> 4 == 14 )
  {
    v4 = a2[2] & 0x3F | ((a2[1] & 0x3F) << 6) | ((*a2 & 0xF) << 12);
    v3 = 3;
  }
  else if ( (signed int)*a2 >> 5 == 6 )
  {
    v4 = a2[1] & 0x3F | ((*a2 & 0x1F) << 6);
    v3 = 2;
  }
  else
  {
    LOBYTE(v4) = *a2;
    v3 = 1;
  }
  *a1 = v4;
  return v3;
}
```

发现对 &unk_402008 的字串进行了复杂的置换，动态调试发现置换后就是对话框输出的内容 I never broke the encoding:

注意到 sub_401160 函数的长度为 0x1C

但是我们发现 &unk_402008 字串明显长得多，计算后长度为 0xAF

![image-20200428142510420](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge9gp8uk1ej30u02wxx6p.jpg)

结合题目 **Overlong**，猜想我们需要将全部字串置换

## OD Patch

用 OD 打开程序，定位到 0x1C 处

![image-20200428142718986](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge9grgaea1j31ds0sidhi.jpg)

单步执行，等 0x1C 载入内存后，修改为 0xAF

![image-20200428142831292](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge9gsp6vbqj30wo0iaaat.jpg)

运行程序

![image-20200428142925823](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge9gtndov4j30z60gw0tc.jpg)

## 得到 flag

`flag{I_a_M_t_h_e_e_n_C_o_D_i_n_g@flare-on.com}`

