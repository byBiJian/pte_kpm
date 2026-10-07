# pte_kpm
基于Arm64针对AndroidGki内核开发的kpm内核模块，理论使用与所有5.4内核及以上APatch/刷入了KPatch Next的设备，pte页表权限断点，内核态消化SIGSEGV理论用户态无感，断点命中后支持修改目标寄存器x0~x8/w0~w8后步进恢复页表权限，详细见README.md，稍后将开源。 无不良引导，开发仅作用于合法安全网络逆向研究，造成一切后果与开发者无关
