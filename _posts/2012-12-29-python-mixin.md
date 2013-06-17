---
layout: post
title: python 中的mixin特性
tags: [python, linux程序设计]
---

python支持多继承后，但能否支持动态继承性质?在程序运行过程中，重定义类的继承，python是支持这种动态继承性质的。这也就是python中的mixin，在定义类过程中改变类的继承顺序，继承类。当某个模块不能修改时，通过mixin方式可以动态添加该类的方法，动态改变类的原有继承体系。弄懂了多继承，mixin特性就简单多了。
但需要注意mixin后的具体继承体系的改变。测试代码在[这里](https://github.com/ytaoWang/test/blob/master/python_test/mixin.py)

参考资料
------------
很详细的MixIn介绍:http://www.linuxjournal.com/node/4540/print


