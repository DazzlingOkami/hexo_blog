---
title: 一个极简的DAPLink
author: WangGaojie
---

平时我的主要兴趣方向都是嵌入式软件开发方面的内容，但偶尔也做一些硬件相关的内容。我的硬件设计水平一般，仅仅是能够用嘉立创的EDA软件设计一些简单的PCB，焊接能力更是拿不出手😅。

之前我偶然间看到一个别人设计的DAPLink，使用51单片机完成的。让我感兴趣的是，如果更换同类型的更小封装芯片，或许可以设计出一个非常小的DAPLink，所以就有了我的一个设计。

我使用CH552E单片机加一个LDO以及其它几个电阻电容就完成了它的主要电路。一个极简的DAPLink，具有一个SWD接口的UART接口，只有大约USBtype-C母座的大小。如果你喜欢，它甚至可以作为你的板载调试器。

![焊接完成的样品](sample.jpeg)

我已经让它工作了很长一段时间，看起来非常好用。在某个量产项目中，我把它当作固件批量下载工具，目前还没有遇到明显的问题。

最后，它的原理图、PCB、源代码、编译好的固件在这里：
原理图PCB：[立创开源-DAPLink](https://oshwhub.com/dazzlingokami/padlink)
固件、源码：https://github.com/DazzlingOkami/CH552-DAPLink
