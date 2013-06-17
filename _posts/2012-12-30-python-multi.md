---
layout: post
title: python多继承
tags: [python]
---

多继承在针对派生类中的同名函数时，python 需要对其进行线性化(C3 Linerization),将继承图关系线性化。通过线性化，再依次查找类方法，直至找到该方法为止。线性化算法是python多继承的核心部分。线性化过程中，必须满足两个性质:

* 单调性

 如果C从C<sup>'</sup>直接继承属性P(如图1),(C != C<sup>'</sup>),C<sup>'</sup>所有的父类(包含C<sup>'</sup>)均在线性化中都是C的子列表。这样的话，C的线性化列表不用遍历C的所有继承体系，直接通过C的直接继承父类线性化列表获得(通过归并父类列表获得)。
 
 ![继承图1](/blog/images/python_multi_1.png)

* 一致性

直接父类的顺序通过用户来声明，父类线性化的顺序为从左到右。在进行线性化归并过程中，一致性主要保证类的局部优先级顺序，它定义了两个变量:<br/>
1) prec<sub>C</sub>:类C的局部顺序</br>
2) prec:局部顺序的并集</br>

类C1,C2,C1 < <sub>prec<sub>C</sub></sub> C2 ===> C1 < <sub>prec</sub> C2,prec 为无环关系图。<br/>
通过上面的两个定义，引申出最终边的关系图。扩展关系<<sub>e</sub>:类线性化列表p1=(c ... a),p2=(c ... b),a < <sub>prec<sub>C</sub></sub> b ===> a <<sub>e</sub> b，这样就获得了最后的类关系图，符合<<sub>e</sub>和原来类关系图的线性化就是类最终的关系化图。

假设类的继承关系图为:

![继承图2](/blog/images/python_multi_2.png)

通过计算扩展关系 <<sub>e</sub>:<br/>
5 <<sub>e</sub> 6  <br/>
2 <<sub>prec<sub>1</sub></sub> 3 <br/>
4 <<sub>prec<sub>2</sub></sub> 5 <br/>
4 <<sub>prec<sub>3</sub></sub> 6 <br/>

这样最终的关系图为:

![继承图3](/blog/images/python_multi_3.png)

线性化关系为:<br/>

    L(7) = {'object'}
    L(4) = {'4','7'}
    L(5) = {'5','7'}
    L(6) = {'6','7'}
    L(2) = {'2','4','5','7','object'}
    L(3) = {'3','4','6','7','object'}
    L(1) = {'1','2','3','4','5','6','7','object'}

python 代码在[这里](https://github.com/ytaoWang/test/blob/master/python_test/multi_inheritance.py)

参考资料:<br/>
R.Ducournau. Proposal for a Monotonic Multiple Inheritance Linearization,OOPSLA'94 <br/>
R.Ducouranu. Monotonic Conflict Resolution Mechanisms for Inheritance,OOPSLA'92 <br/>
Kim Barrett. A Monotonic Superclass Linearization for Dylan <br/>
python 多继承:http://www.artima.com/weblogs/viewpost.jsp?thread=246488 <br/>
