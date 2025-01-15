---
title: 一个整数无法正常溢出导致的bug
author: WangGaojie
---

lvg是一款非常流行的图形库，最近在使用它的过程中发现一个有意思的问题，这里做简单记录。
话说事情是这样的，我为按钮设计了一个动画，大概是按钮一直沿y轴周期性的往复移动。

```c
lv_anim_t by;
lv_anim_init(&by);
lv_anim_set_exec_cb(&by, (lv_anim_exec_xcb_t) lv_obj_set_y);
lv_anim_set_var(&by, btn);
lv_anim_set_time(&by, 1797);
lv_anim_set_values(&by, 2, 106);
lv_anim_set_playback_time(&by, 1797);
lv_anim_set_repeat_count(&by, LV_ANIM_REPEAT_INFINITE);
lv_anim_start(&by);
```

代码非常简单，就是为btn绑定一个无限重复次数的动画。代码刚开始可以好好运行，但是动画执行了几分钟后这个按钮的动画突然暂停了。按钮保持在了一个固定的位置，我不明白发生了什么。我可以确认系统内各个任务都在正常的执行，图形页面上的一些效果都在正常显示，就只有这个按钮的动画仿佛被删除了一样。

系统重启了几次，这个bug视乎可以一直复现，我决定连着调试器看看，到底发生了什么，我在关键的一些的地方设置了断点，等待故障发生。奇怪的是故障发生后没有命中任何断点，我立即暂停了程序运行，我决定进入到动画处理的定时器回调函数中看看，相关的动画句柄数据到达是什么样的。

在anim_timer()函数内设置一个断点，成功命中，说明动画处理相关的定时器没有被删除，但是动画执行的回调却不执行。检查lv_anim_t结构体内的数据，发现a->act_time为一个很大的负数值，这就看起来很离谱，这意味着距离下一次动画被激活还有很长的时间要等待，动画效果因此被暂停。为什么act_time是负数了。

```c
uint32_t elaps = lv_tick_elaps(a->last_timer_run);
a->act_time += elaps;
```

在我将能够排除的地方都排除后，我将问题点定位到了上面这里。elaps的计数值肯定是一个正数，但如果elaps非常大的话可能导致后面的累加计算溢出导致计算出的值为负数。

lv_tick_elaps()的作用是计算当前时刻距离last_timer_run时刻的时间差，它的实现如下：

```c
uint32_t lv_tick_elaps(uint32_t prev_tick)
{
    uint32_t act_time = lv_tick_get();

    /*If there is no overflow in sys_time simple subtract*/
    if(act_time >= prev_tick) {
        prev_tick = act_time - prev_tick;
    }
    else {
        prev_tick = UINT32_MAX - prev_tick + 1;
        prev_tick += act_time;
    }

    return prev_tick;
}
```

看到这里，我突然明白了问题点，lv_tick_elaps函数内做了整数溢出处理，但是它默认从lv_tick_get()返回的时间计数周期为UINT32_MAX。如果时间周期不是UINT32_MAX，这就导致溢出计算错误。

再看看lv_tick_get()函数的实现，它最终会调用由void lv_tick_set_cb(lv_tick_get_cb_t cb)设置的回调函数，我这里是：

```c
static uint32_t get_gui_tick(void){
    return xTaskGetTickCount() * 1000u / configTICK_RATE_HZ;
}

// lv_tick_set_cb(get_gui_tick);
```

我这里的configTICK_RATE_HZ=20000，这就导致get_gui_tick函数返回的周期比UINT32_MAX小了20倍。所以导致程序运行时act_time计算错误。

lvgl似乎没有对用户导入的时间查询回调函数做一定的规范，导致了该问题的发生。

修改FreeRTOS内核配置，将configTICK_RATE_HZ设置为1000，这样可以解决问题吗？哈哈，如果这样想你很快就会调入另外一个陷阱。

```c
xTaskGetTickCount() * 1000u / configTICK_RATE_HZ
```

先乘法再除法会导致xTaskGetTickCount()时间的周期缩小1000倍，所以不要忽视这些细节。

我这里为了不修改FreeRTOS内核配置，所以我决定使用一个硬件定时器来彻底解决该问题，确保导入到图形库内的时间查询函数返回值的周期为UINT32_MAX。

这是第一次遇到由于整数溢出边界不严谨导致的故障。如果你有类似的困扰可以将configTICK_RATE_HZ设置为1000并调用lv_tick_set_cb(xTaskGetTickCount)来解决该问题。
