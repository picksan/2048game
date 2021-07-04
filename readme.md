# c语言实现2048游戏

### 一、实验介绍

#### 1.1 实验内容

通过本次课程我们将使用 C 语言实现一个命令行版本的 *2048* 小游戏。所以这需要你有 C 语言基础，不需要非常精通，懂得基本语法即可。

#### 1.2 实验知识点

- C 语言语法
- 游戏设计与实现思路
- 屏幕绘图库`ncurses`的使用

#### 1.3 实验环境

- Xfce 终端（Xfce Terminal）：Linux 命令行终端，打开后会进入 Bash 环境，可以用来执行 Linux 命令。
- GVim：非常好用的编辑器，不会使用的可以参考课程 [《Vim 编辑器》](https://www.lanqiao.cn/courses/2)。

#### 1.4 适合人群

本课程适合有 C 语言基础，希望在动手能力上得到提升的同学，熟悉模块和主流程的运行流程。

#### 1.5 代码获取

```bash
wget https://labfile.oss.aliyuncs.com/courses/155/game_2048.c
# 编译
gcc game_2048.c -o 2048 -lcurses
# 执行
./2048
```

#### 1.6 效果图

最终效果图是这样的： ![此处输入图片的描述](https://doc.shiyanlou.com/document-uid59274labid452timestamp1477047436903.png)

### 二、实验原理

#### 2.1 基础知识

要实现我们的 2048 小游戏，需要涉及一些数据结构的知识，以及一些 Linux 的系统调用。此外，为了方便在屏幕上使用字符绘图，我们还需要使用一个文本界面的屏幕绘图库 `ncurses` ，具体到操作就是在编译的时候需要加上 `-lcurses` 选项。

`ncurses` 库的安装操作如下：

```bash
sudo apt-get install libncurses5-dev
```

这个库在实验环境中已经安装了：

![此处输入图片的描述](https://doc.shiyanlou.com/document-uid600404labid452timestamp1550654748226.png)

#### 2.2 设计思路

要实现 2048 游戏目前有两个关键点：

1. 在满足条件情况下消除方块
2. 允许在游戏主界面（16 宫格）中任意一格输出数据

其中第二点借助 `ncurses` 库可以较容易实现，但是第一点要稍微麻烦些。

第一点的实现思路是，我们创建一个与游戏地图相同维数的数组矩阵，通过数组矩阵来维护 2048 游戏中每个格子的数据与状态，从而玩家的移动操作都可以映射为对数组矩阵的操作。

### 三、实验步骤

#### 3.1基础工作

进入 `/home/shiyanlou/Code/` 目录中创建项目文件 `game_2048.c` 。使用 gedit 打开该文件编写代码。

![此处输入图片的描述](https://doc.shiyanlou.com/document-uid242676labid452timestamp1475041061262.png)

让我们首先来完成一些基础工作，首先是引入头文件：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <curses.h>
#include <unistd.h>
#include <signal.h>
#include <time.h>
```

再创建几个全局变量：

```cpp
// 游戏主界面是一个 4*4 的 16 宫格，使用二维数组进行表示，用 0 表示空格
int a[4][4] = {0};
// 16 宫格中空格的个数
int empty;
int old_y, old_x;
```

为了调用起来方便，先声明一下所需要创建的函数原型:

```cpp
void draw();  // 用于绘制游戏界面
void play();  // 游戏运行的逻辑主体
void init();  // 初始化函数，用于完成一些必要的初始化操作
void draw_one(int y, int x);  // 绘制单个数字
void cnt_value(int *new_y, int *new_x);
int game_over();  // 结束游戏
int cnt_one(int y, int x);
```

#### 3.2main函数

先看看代码：

```cpp
int main()
{
    init();
    play();
    endwin();

    return 0;
}
```

首先调用 `init` 进行初始化一些关键参数，然后调用 `play` 函数运行游戏主题，然后再调用 `endwin` 关闭游戏窗口。

下面看一下 `init` 函数：

```cpp
void init()
{
    int x, y;

    initscr();
    cbreak();
    noecho();
    curs_set(0);

    empty = 15;
    srand(time(0));
    x = rand() % 4;
    y = rand() % 4;
    a[y][x] = 2;
    draw();
}
```

`init` 函数首先初始化屏幕，并且随机生成两个数字 `x, y` 用于指定方格的位置坐标，并给数组 `a[][]` 中相应位存入数字 2 ，然后再调用 `draw` 函数绘制对应的字符界面。

这部分用于初始化游戏界面，使得刚进入游戏的时候界面如下：

![此处输入图片的描述](https://doc.shiyanlou.com/document-uid242676labid452timestamp1475045327967.png)

然后编写绘制方格的函数，draw 函数代码如下：

```cpp
void draw()
{
    int n, m, x, y;
    char c[4] = {'0', '0', '0', '0'};
    clear();

    for(n = 0; n < 9; n += 2)   //绘制横线，n代表行，m代表列
    {
        for(m = 0; m < 21; m++)
        {
            move(n, m);
            addch('-');
            refresh();
        }
    }
    for(m = 0; m < 22; m += 5)  //绘制竖线
    {
        for(n = 1; n < 8; n++)
        {
            move(n, m);
            addch('|');
            refresh();
        }
    }
    for(y = 0; y < 4; y++)     //绘制数字
    {
        for(x = 0; x < 4; x++)
        {
            draw_one(y, x);
        }
    }
}
```

其中 `draw_one` 函数用于绘制单个数字，其定义如下：

```cpp
void draw_one(int y, int x)
{
    int i, m, k, j;
    char c[5] = {0x00};
    i = a[y][x];
    m = 0;
    while(i > 0)
    {
        j = i % 10;
        c[m++] = j + '0';
        i = i / 10;
    }
    m = 0;
    k = (x + 1) * 5 - 1;
    while(c[m] != 0x00)
    {
        move(2*y+1, k);
        addch(c[m++]);
        k--;
    }
}
```

#### 3.3编写play函数

然后来看看游戏运行的核心 `play` 函数。

我们将通过 `W, S, A, D` 和方向键来分别控制上、下、左、右方向的移动，按 `Q` 键可以退出界面。这次的 `play` 函数相对较长，希望大家耐着性子看下去。其实总体结构很简单，只是过程比较繁琐。

```cpp
void play()
{
    int x, y, i, new_x, new_y, temp;
    int old_empty, move;
    char ch;

    while(1)
    {
        move = 0;
        old_empty = empty;
        ch = getch();
        switch(ch) {
            case 97:    //左移  a
            case 104:   // h
            case 68:    // 左移方向键
                for(y = 0; y < 4; y++) //合并相同的数字
                    for(x = 0; x < 4; )
                    {//在x位置的右边，找到一个有数字的位置，如果数字一样，则合并，不一样，就跳到这个x
                        //然后在新x的右边继续找数字，直到遍历完x
                        if(a[y][x] == 0)
                        {
                            x++;
                            continue;
                        }
                        else
                        {
                            for(i = x + 1; i < 4; i++)
                            {
                                if(a[y][i] == 0)
                                {
                                    continue;
                                }
                                else
                                {
                                    if(a[y][x] == a[y][i])
                                    {
                                        a[y][x] += a[y][i];
                                        a[y][i] = 0;
                                        empty++;
                                        break;
                                    }
                                    else
                                    {
                                        break;
                                    }
                                }
                            }
                            x = i;
                        }
                    }
                for(y = 0; y < 4; y++) //把数字向左移动，把0填充掉
                    for(x = 0; x < 4; x++)
                    {//遍历x，如果不是x不是0，x的左边是0，则把x的数字左移，直到左边有数字或者是第0位
                        if(a[y][x] == 0)
                        {
                            continue;
                        }
                        else
                        {
                            for(i = x; (i > 0) && (a[y][i-1] == 0); i--)
                            {
                                a[y][i-1] = a[y][i];
                                a[y][i] = 0;
                                move = 1;
                            }
                        }
                    }
                break;
            case 100:   //右移 d
            case 108:   // l
            case 67:    //右移方向键
                for(y = 0; y < 4; y++)
                    for(x = 3; x >= 0; )
                    {
                        if(a[y][x] == 0)
                        {
                            x--;
                            continue;
                        }
                        else
                        {
                            for(i = x - 1; i >= 0; i--)
                            {
                                if(a[y][i] == 0)
                                {
                                    continue;
                                }
                                else if(a[y][x] == a[y][i])
                                {
                                    a[y][x] += a[y][i];
                                    a[y][i] = 0;
                                    empty++;
                                    break;
                                }
                                else
                                {
                                    break;
                                }
                            }
                            x = i;
                        }
                    }
                for(y = 0; y < 4; y++)
                    for(x = 3; x >= 0; x--)
                    {
                        if(a[y][x] == 0)
                        {
                            continue;
                        } else
                        {
                            for(i = x; (i < 3) && (a[y][i+1] == 0); i++)
                            {
                                a[y][i+1] = a[y][i];
                                a[y][i] = 0;
                                move = 1;
                            }
                        }
                    }
                break;
            case 119:   //上移 w
            case 107:   //k
            case 65:    //上移方向键
                for(x = 0; x < 4; x++)
                    for(y = 0; y < 4; )
                    {
                        if(a[y][x] == 0)
                        {
                            y++;
                            continue;
                        }
                        else
                        {
                            for(i = y + 1; i < 4; i++)
                            {
                                if(a[i][x] == 0)
                                {
                                    continue;
                                }
                                else if(a[y][x] == a[i][x])
                                {
                                    a[y][x] += a[i][x];
                                    a[i][x] = 0;
                                    empty++;
                                    break;
                                } else
                                {
                                    break;
                                }
                            }
                            y = i;
                        }
                    }
                for(x = 0; x < 4; x++)
                    for(y = 0; y < 4; y++)
                    {
                        if(a[y][x] == 0)
                        {
                            continue;
                        }
                        else
                        {
                            for(i = y; (i > 0) && (a[i-1][x] == 0); i--)
                            {
                                a[i-1][x] = a[i][x];
                                a[i][x] = 0;
                                move = 1;
                            }
                        }
                    }
                break;
            case 115:   //下移 s
            case 106:   //j
            case 66:    //下移方向键
                for(x = 0; x < 4; x++)
                    for(y = 3; y >= 0; )
                    {
                        if(a[y][x] == 0)
                        {
                            y--;
                            continue;
                        }
                        else
                        {
                            for(i = y - 1; i >= 0; i--)
                            {
                                if(a[i][x] == 0)
                                {
                                    continue;
                                }
                                else if(a[y][x] == a[i][x])
                                {
                                    a[y][x] += a[i][x];
                                    a[i][x] = 0;
                                    empty++;
                                    break;
                                }
                                else
                                {
                                    break;
                                }
                            }
                            y = i;
                        }
                    }
                for(x = 0; x < 4; x++)
                    for(y = 3; y >= 0; y--)
                    {
                        if(a[y][x] == 0)
                        {
                            continue;
                        }
                        else
                        {
                            for(i = y; (i < 3) && (a[i+1][x] == 0); i++)
                            {
                                a[i+1][x] = a[i][x];
                                a[i][x] = 0;
                                move = 1;
                            }
                        }
                    }
                break;
            case 'Q':
            case 'q':
                game_over();
                break;
            default:
                continue;
                break;
        }
        if(empty <= 0) //没有空格，游戏结束
            game_over();
        if((empty != old_empty) || (move == 1))
        {//移动过或者合成过就产生新的数字，新数字只能在0的上产生，数字大小位2或者4
            do{
                new_x = rand() % 4;
                new_y = rand() % 4;
            }while(a[new_y][new_x] != 0);

            cnt_value(&new_y, &new_x);

            do {
                temp = rand() % 4;
            }while(temp == 0 || temp == 2);
            a[new_y][new_x] = temp + 1;//生成新的 2 4随机数
            empty--;
        }
        draw();
    }
}
```

#### 3.4其他部分

下面的函数用于生成新数字的位置，判断方法比较简单。

```cpp
// 统计(y, x)对应的格子周围一圈的空格的个数
int cnt_one(int y, int x)
{
    int value = 0;

    if(y - 1 > 0)
        a[y-1][x] ? 0 : value++;
    if(y + 1 < 4)
        a[y+1][x] ? 0 : value++;
    if(x - 1 >= 0)
        a[y][x-1] ? 0 : value++;
    if(x + 1 < 4)
        a[y][x+1] ? 0 : value++;
    if(y - 1 >= 0 && x - 1 >= 0)
        a[y-1][x-1] ? 0 : value++;
    if(y - 1 >= 0 && x + 1 < 4)
        a[y-1][x+1] ? 0 : value++;
    if(y + 1 < 4 && x - 1 >= 0)
        a[y+1][x-1] ? 0 : value++;
    if(y + 1 < 4 && x + 1 < 4)
        a[y+1][x+1] ? 0 : value++;

    return value;
}

void cnt_value(int *new_y, int *new_x)
{
    int max_x, max_y, x, y, value;
    int max = 0;

    max = cnt_one(*new_y, *new_x);
    for(y = 0; y < 4; y++)
        for(x = 0; x < 4; x++)
        {
            // 如果(y, x)对应的空格为空
            if(!a[y][x])
            {
                // 优先选取周围空格最多的空格展示新数字
                value = cnt_one(y, x);
                if(value > max && old_y != y && old_x != x)
                {
                    // 避免在同一位置反复出现新数字
                    *new_y = y;
                    *new_x = x;
                    old_x = x;
                    old_y = y;
                    break;
                }
            }
        }
}
```

游戏结束子函数。

```cpp
int game_over()
{
    sleep(1);
    endwin();
    exit(0);
}
```

代码介绍到此结束，完成源代码文件编写 `/home/shiyanlou/Code/game_2048.c`。

#### 3.5编译

接下来可以开始进行编译和运行，编译的命令如下：

```bash
gcc game_2048.c -o 2048 -lcurses
```

运行：

```bash
./2048
```

![此处输入图片的描述](https://doc.shiyanlou.com/document-uid242676labid452timestamp1475041744146.png)

运行效果如下：

![此处输入图片的描述](https://doc.shiyanlou.com/document-uid242676labid452timestamp1475042753743.png)

#### 四、实验总结

到此，我们的 2048 游戏就完成了。我们了解了一个游戏应该如何编写与运行，学习了绘图库`ncurses`的使用。如果 你愿意，可以将简陋的 ASCII 字符换成漂亮的图片，再加上积分牌等等。总之，程序会随着你的想象力越长越大。