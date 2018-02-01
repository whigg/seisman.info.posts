---
title: GMT 绘制地理坐标与笛卡尔坐标混合体
author: SeisMan
date: 2014-04-26
lastmod: 2015-06-29
categories:
  - GMT
tags:
  - GMT技巧
slug: mix-geographical-coordinate-with-cartesian-coordinate
---

## 提出问题

需要画一个沿着某个经线或纬线的深度剖面图，即 X 轴表示经度或纬度，Y 轴表示深度，如下图：

![](/images/2014042601.png)

<!--more-->

## 分析问题

这张图比较特殊的地方在于，X 轴是地理坐标，Y 轴是笛卡尔坐标。最简单的办法是直接用线性投影 `-JX` 来绘制整张图，比如:

    gmt psbasemap -R40/50/0/600 -JX15c/-10c -Bx2 -By100 > mix.ps

效果如下图：

![](/images/2014042602.png)

显然，这张图还差了一点，X 轴的坐标都没有 “度” 符号以及 `WSEN` 后缀，无法体现出其地理坐标的特征。

## 解决问题

解决办法如下:

    gmt set FORMAT_GEO_MAP +ddd:mmF
    gmt psbasemap -R40/50/0/600 -JX15cd/-10c -Bx2 -By100 > mix.ps

几点说明：

-   `FORMAT_GEO_MAP` 用于设置地理坐标的显示方式。这里的 `+ddd:mmF` 表示以 `度:分` 的形式显示，并加上 `EWSN` 后缀；
-   `-JX15cd/-10c` 设定了线性投影方式。X 轴多了一个 `d`，用于显式指定 X 轴为地理坐标；Y 轴没有 `d`，为正常的笛卡尔坐标；这里 Y 轴的长度是负值，表示 Y 值从上到下递增，以符合常见的深度的定义；
-   GMT4 同理，但 GMT 4.5.13 似乎存在 bug，当使用 `-JX15cd/-10c` 时，X 轴的标注的位置会出现偏差；尚未发布的 GMT 4.5.14 中该 bug 已被修复；
-   X 轴由于是地理坐标，所以不能用 `-B` 选项给 X 轴加标题；如果需要加标题的话，只能使用 `pstext`；

## 其它解决方法

下面列出最初对于这个问题的分析以及两种稍复杂的解决方案。虽说下面的两种方案更复杂了，但其思路与想法还是很有意思的，或许在其它问题上可以借鉴，因而将其列出。

### 分析问题

1.  因为 Y 轴是线性坐标，所以必然只能选择线性投影，即 `-JX`
2.  线性投影的情况下，图的主体很简单，关键在于 X 轴坐标的 “度” 符号以及后缀 E 上
3.  尝试将 X 轴和 Y 轴都当作线性坐标，然后对于 X 轴设置其 **单位** 为特殊的 “度” 符号。此法看似可行，但实际上 GMT 内部设置了单位与标注之间的距离，通过单位设置的 “度” 符号明显离标注的距离较远，不太美观。
4.  为了使 X 轴有一个 “度” 符号，X 轴必须当作地理坐标处理；而 Y 轴的范围已经超过了地理坐标的范围，所以必须当作线性坐标处理；
5.  综上，必须使用两个命令来绘制边框，分别绘制地理坐标的 X 轴（`-BxxxSN`）和线性坐标的 Y 轴（`-BxxxEW`）；
6.  关于如何绘制地理坐标的 X 轴，有两种解决办法。

### 解决问题

#### 法 1

``` bash
#/bin/bash
Rx=40/50    # X 轴范围
Ry=0/600    # Y 轴范围
R=$Rx/$Ry
B=2/100     # 间隔
J=X15c/10c
PS=map.ps

psxy -R$R -J$J -T -K > $PS   # 写入 PS 文件头

psbasemap -R$R -J$J -B${B}SN -K -O --D_FORMAT='%g\260E' >> $PS    # 绘制 X 轴
psbasemap -R$R -J$J -B${B}EW -K -O >> $PS     # 绘制 Y 轴

# 这里放置其它绘图命令，不再使用 B 选项

psxy -R$R -J$J -T -O >> $PS  # 写入 PS 文件尾
rm .gmt*
```

这里绘制 X 轴时直接使用 `--D_FORMAT=%g\260E` ，使得在该命令中 `D_FORMAT` 的值为 `%g\260E`，
即设置显示浮点数时在其后加上 “度” 符号以及后缀“E”。

此法的优点在于简单，缺点在于后缀 “E” 是固定值，无法处理东西经同时存在的情况。

#### 法 2

``` bash
#/bin/bash
Rx=40/50    # X 轴范围
Ry=0/600    # Y 轴范围
Rfake=0/1   # 假轴范围
R=$Rx/$Ry
B=2/100     # 间隔
J=X20c/15c
PS=map.ps

psxy -R$R -J$J -T -K > $PS   # 写入 PS 文件头

psbasemap -Rg$Rx/$Rfake -J$J -B${B}SN -K -O --BASEMAP_TYPE=plain >> $PS    # 绘制 X 轴
psbasemap -R$R -J$J -B${B}EW -K -O >> $PS     # 绘制 Y 轴

# 这里放置其它绘图命令，不再使用 B 选项

psxy -R$R -J$J -T -O >> $PS  # 写入 PS 文件尾
rm .gmt*
```

### 其它的说明

1.  GMT 的 B 选项提供了这样一个功能，可以使用形如 `-Bgxmin/xmax/ymin/ymax` 的语法，其中 `g` 告诉命令即便使用 `-JX` 投影，也认为其是地理坐标。由于是地理坐标，“度”符号以及后缀 “E” 就很容易出来了
2.  使用 `-Bgxmin/xmax/ymin/ymax` 存在两个问题
    1.  虽然是线性投影，但是由于使用了地理坐标，GMT 会默认将底图类型设置为 fancy。这一点可以设置 `BASEMAP_TYPE` 等于 `plain` 来解决。
    2.  Y 轴被当作地理坐标，所以 ymin 和 ymax 的范围被限制在 [-90,90] 之内

3.  在此例中在绘制 X 轴时引入了一个假的 Y 轴 `0/1`，以满足 `-Rgxmin/xmax/ymin/ymax` 形式中对 ymin 和 ymax 范围的限制。

这样，X 轴和 Y 轴就都设计好了，接下来要做的就只是保证其它命令都不使用 B 选项即可。

## 修订历史

-   2014-04-26：初稿；
-   2014-04-26：修改脚本，解决了对 Y 轴范围的限制；Thanks to Chen Zhaohui；
-   2014-06-09：通过修改 `D_FORMAT` 以解决地理坐标的度符号；该方法由刘珠妹提供；
-   2014-07-09：找到了一种非常简单的方法来解决该问题；
-   2014-11-24：修正了 `--PLOT_DEGREE_FORMAT` 中的小问题；
-   2015-06-29：GMT4.5.13 在解决该问题时有其他 bug，这里使用 GMT5；