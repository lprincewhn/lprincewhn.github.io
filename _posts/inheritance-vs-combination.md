---
layout: post
title: 继承和组合
key: 20160729
tags: OOP inheritance combination
modify_date: 2016-09-25
---

记得当年在大学里面学习面向对象编程的时候，印象最深刻的就是继承了，做各种各样的编程实验把继承，重载，覆盖，隐藏各种机制玩坏了，一度让我由“不继承，无对象”的错误印象。但后来学习设计模式时候又被告诉“组合优于继承”，当第一次听到这个说法的时候，我的第一反应是“Are you kidding”？今天回想起来，于是就想聊聊继承和组合那些事。

<!--more-->

## 1. 什么叫做继承，什么叫做组合？
简单来说，继承是is A的关系，组合式has A的关系。听起来好像简单直接，明了易懂，但是听完之后，你真的懂了吗？

- 父亲和儿子是继承关系吗？
- 长方形和正方形是继承关系吗？
- 动物和人是继承关系吗？

上述三个例子是在教材讲解继承的时候最常使用的例子，但这个问题的答案却是不确定的。为什么？因为他取决于业务场景。

其实，要真正理解他们的关系，还是要回到“面向对象”的本质说起，即：数据与行为绑定。数据与行为绑定。数据与行为绑定。重要的事情说三次，这不单单是重要的事情，这是进行面向对象编程时最核心最本质的思维方式。
讨论面向对象中的任何机制都绕不开这个本质。

继承中的is指的是：父类中的一切行为，子类都可以支持。所以上面的问题等价于：
- 父亲能做的事情儿子都能做吗？ 不能。父亲可以上班赚钱，儿子不能。
- 长方形能做的事情正方形都能做吗？ 不能。长方形可以单独修改宽或者高，正方形不行。
- 动物能做的事情人都能做吗？不能。有些动物会有用，但有些人不会。

但是我们写出来的对象所在的业务场景不可能涵盖对象的所有行为和属性，当业务场景不涉及上述例外行为的时候，那上面三个继承关系很可能是可以成立的。比如说在长方形和正方形的例子中，如果我们的业务场景只涉及计算周长，面积等通用行为