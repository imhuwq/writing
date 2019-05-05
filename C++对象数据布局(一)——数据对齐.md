---
title: C++对象数据布局(一)——数据对齐的陷阱
date: 2019-05-05 12:07:35
categories:
- 技术
tags:
- c++
---

C++ Class 对象的数据布局和 C Struct 数据布局遵循同样的原则，按顺序排布并考虑内存对齐的要求。  
但是 C++ Class 对象相比于 C Struct 有其创新之处。C++ Class 添加了两个新的 access section，支持在类内声明函数，最重要的是，添加了“继承”的特性。  
这其中有什么可怕的陷阱吗？直接公布答案有什么意思，你得自己一步一步趟过去才行。  
小心，别中招了！  
<!--more-->

## 零、一些准备
为了更好地探查 C++ 对象地数据排布，我替你准备了一些辅助的宏用作拐杖：  
``` c++
// 获取两个指针的地址差值
inline int diff(void *a, void *b) { return ((char *) a - (char *) b); }

// 接下来的几个宏用来打印一个表格，这个表格里面包含了 Class 里每个数据的 Offset 和大小
// 首先打印一个 Class 的名字和总大小，然后再打印一行 “SubObject ::Attribute : Offset(~Size) 0xAddres” 用作表头
#define PRINT_START(cls) printf("%s Layout (Total Size %03ld):\n%-20s::%-20s: %s\n", \
                                #cls, sizeof(cls), "SubObject","Attribute","Offset(~Size) 0xAddress")
// 打印 obj->attr 的 Offset 和大小，obj 是一个 cls 类的指针
// 此处的原理是：obj->attr 地址等于 obj 地址加上 attr 的 offset，所以 obj->attr - obj 得到 attr 的 offset
#define PRINT_OFFSET(cls, obj, attr) printf("%-20s::%-20s: %06d(~%04ld) %p\n",\
                                            cls, #attr, diff(attr, obj),sizeof(*attr), (void *)attr)
// 打印表格的结尾分隔符
#define PRINT_END() printf("==========\n\n")
```

## 一、简单 Class
你决定先从一个简单的 Class 开始热身:  
``` c++
class C1 {
public:
    int val1;
    char bit1;

    static void DisplayLayout(C1 *obj) {
        PRINT_START(C1);
        PRINT_OFFSET("C1", obj, &obj->val1);
        PRINT_OFFSET("C1", obj, &obj->bit1);
        PRINT_OFFSET("C1", obj, &obj->bit1_);
        PRINT_OFFSET("C1", obj, &obj->bit1_2_);
        PRINT_END();
    }

private:
    char bit1_;
    char bit1_2_;
};


int main() {
    C1 c1;
    C1::DisplayLayout(&c1);
}

```
执行这个程序会得到如下的输出：  
```
C1 Layout (Total Size 008):
SubObject           ::Attribute           : Offset(~Size) 0xAddress
C1                  ::&obj->val1          : 000000(~0004) 0x7fff1ec2b380
C1                  ::&obj->bit1          : 000004(~0001) 0x7fff1ec2b384
C1                  ::&obj->bit1_         : 000005(~0001) 0x7fff1ec2b385
C1                  ::&obj->bit1_2_       : 000006(~0001) 0x7fff1ec2b386
==========
```
你甚至把它画成了图：  
![Padding-C1-Declaration-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1-declaration-layout.png)

指着这幅图，你说，“4 个数据总共占据了 7 个 bytes，其中 `val1` 作为 int 类型占用了 4 个，对齐要求用掉了 1 个，而函数 `DisplayLayout` 没有占用对象的空间。public 和 private 的数据是紧密排布在一起的，不同的 section 之间不会出现 padding，而每个 section 内部的数据按照声明顺序都排布在一起”。  
于是，你大声宣布：“C++ Class 的 access section 和类内函数没有数据对齐陷阱，二者都不会对 C++ Class 的数据布局造成异于 C Struct 的结果”！  
高手果然是高手，一个简单的 Class 就探明了两个方向的细节。

## 二、组合 Class
好的，那继承 Class 的情况呢？  
“慢着”，你说，“我们先看看组合的情况”。多年的 Debug 功力和直觉告诉你，所有的“显而易见”都必须“眼见为实”，C++ Class 和 C Struct 的组合理论上遵循同样的原则，但你还是要亲自确认一下。
你再定义了一个基于 `C1` 的组合 Class:  
![Padding-C12-Declaration](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c12-declaration.png)  

C12 的数据布局是怎么样的呢？程序验证如下：  
``` c++
class C12;

class C1 {
    friend C12;
public:
    int val1;
    char bit1;

    static void DisplayLayout(C1 *obj) {
        PRINT_START(C1);
        PRINT_OFFSET("C1", obj, &obj->val1);
        PRINT_OFFSET("C1", obj, &obj->bit1);
        PRINT_OFFSET("C1", obj, &obj->bit1_);
        PRINT_OFFSET("C1", obj, &obj->bit1_2_);
        PRINT_END();
    }

private:
    char bit1_;
    char bit1_2_;
};

class C2{
public:
    char bit2;
};

class C12 {
public:
    C1 c1;
    C2 c2;

    static void DisplayLayout(C12 *obj) {
        PRINT_START(C12);
        PRINT_OFFSET("C1", obj, &obj->c1.val1);
        PRINT_OFFSET("C1", obj, &obj->c1.bit1);
        PRINT_OFFSET("C1", obj, &obj->c1.bit1_);
        PRINT_OFFSET("C1", obj, &obj->c1.bit1_2_);
        PRINT_OFFSET("C2", obj, &obj->c2.bit2);
        PRINT_END();
    }
};

int main() {
    C12 c12;
    C12::DisplayLayout(&c12);
}
```

输出：  
```
C12 Layout (Total Size 012):
SubObject           ::Attribute           : Offset(~Size) 0xAddress
C1                  ::&obj->c1.val1       : 000000(~0004) 0x7ffd037d36d0
C1                  ::&obj->c1.bit1       : 000004(~0001) 0x7ffd037d36d4
C1                  ::&obj->c1.bit1_      : 000005(~0001) 0x7ffd037d36d5
C1                  ::&obj->c1.bit1_2_    : 000006(~0001) 0x7ffd037d36d6
C2                  ::&obj->c2.bit2       : 000008(~0001) 0x7ffd037d36d8
==========
```

当然，大家都是接受过良好 C 编程基础教育的人，没有人会天真地以为在 C12 中 C2 会直接使用 C1 padding 的那一个 bit：  
![Padding-C12-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c12-layout.png)  

左边的布局虽然是一个错误，但在你看来根本和“陷阱”二字搭不上边。甚至不用代码验证你也能指出，当 C1 的 val1 为 int* 类型时，C1 的总大小将从 8 字节膨胀到 16 字节，而 C12 将从 12 字节膨胀到 24 字节：  
![Padding-C12-Declaration-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c12-declaration-layout.png)  

内存对齐嘛，不是什么新奇玩意儿。再加一个 Class 也能轻松搞定：  
![Padding-C123-Declaration-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c123-declaration-layout.png)  

简而言之，组合 Class 中的数据布局相当于**拼接每个 Class** 的数据布局，并且 Class 和 Class 之间的 **padding 不能重复使用**。最后，组合 Class 也要**对齐最宽的子数据**类型。而这些，也和 C Struct 的数据对齐行为一致。  
也就是说，C++ Class 组合也没有数据对齐陷阱！  

## 三、继承 Class
“好了，现在我们可以检查继承的情况了”。你边说边画，声音沉着而又带着几分得意，笔画灵动，简单几笔就勾勒出了一副 UML 图和预期的数据布局图，让你的小弟去帮你写代码验证。  
![Padding-C1-C2-Declaration-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2-declaration-layout.png)  

“从数据布局的角度说，继承和组合类似，也是在 Class 里面包含了父 Class。在排布完父 Class 后，再开始排布子 Class 自己的 member data。但是一定要明白，父 Class 和子 Class 的 member data 之间的 padding 是不能覆盖的”，你推了推眼镜说着，然后谦虚地建议到：“你应该去看看 *Inside the C++ Object Model* 这本书，里面有详尽地论述。看完之后也就对各种所谓地陷阱一目了然了”。  
说着，小弟就递交了代码和执行结果，和你的预期完全一模一样：  
``` c++
class C1 {
public:
    int* val1;
    char bit1;
};

class C2: public C1{
public:
    char bit2;

    static void DisplayLayout(C2 *obj) {
        PRINT_START(C2);
        PRINT_OFFSET("C1", obj, &obj->val1);
        PRINT_OFFSET("C1", obj, &obj->bit1);
        PRINT_OFFSET("C2", obj, &obj->bit2);
        PRINT_END();
    }
};

int main() {
    C2 c2;
    C2::DisplayLayout(&c2);
}
```

输出：  
```
C2 Layout (Total Size 024):
SubObject           ::Attribute           : Offset(~Size) 0xAddress
C1                  ::&obj->val1          : 000000(~0008) 0x7ffdb7886820
C1                  ::&obj->bit1          : 000008(~0001) 0x7ffdb7886828
C2                  ::&obj->bit2          : 000016(~0001) 0x7ffdb7886830
==========
```

“嗯...”，你看着这个结果，尽量保持着沉思的姿态，好像一副还在谨慎地考虑某种未知情况，虽然你在心里面已经认为万事大吉了。你说，“我们可以考虑再多一层继承的情况”：  
![Padding-C1-C2-C3-Declaration-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2_c3-declaration-layout.png)  

可是小弟马上跟你反馈，C3 的总大小只有 24 字节：  
``` c++
class C1 {
public:
    int* val1;
    char bit1;
};

class C2: public C1{
public:
    char bit2;
};

class C3: public C2{
public:
    char bit3;

    static void DisplayLayout(C3 *obj) {
        PRINT_START(C3);
        PRINT_OFFSET("C1", obj, &obj->val1);
        PRINT_OFFSET("C1", obj, &obj->bit1);
        PRINT_OFFSET("C2", obj, &obj->bit2);
        PRINT_OFFSET("C3", obj, &obj->bit3);
        PRINT_END();
    }
};

int main() {
    C3 c3;
    C3::DisplayLayout(&c3);
}
```

输出：  
```
C3 Layout (Total Size 024):
SubObject           ::Attribute           : Offset(~Size) 0xAddress
C1                  ::&obj->val1          : 000000(~0008) 0x7ffcccc50590
C1                  ::&obj->bit1          : 000008(~0001) 0x7ffcccc50598
C2                  ::&obj->bit2          : 000016(~0001) 0x7ffcccc505a0
C3                  ::&obj->bit3          : 000017(~0001) 0x7ffcccc505a1
==========
```
其图片展示如下：  
![Padding-C1-C2-C3-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2_c3-layout.png)  

“嗯...看起来多级继承下的情况和想象中有点出入，C3 紧接着 C2 开始布局，C2 和 C3 之间 7 个字节的 padding 被 C3 直接使用了”，你冷静地说到，尽量掩饰内心的惭愧，也为刚才的“沉思”感到庆幸，“看来 *Inside the C++ Object Model* 也说得不太正确，Lippman 也有错的时候哇”。  
“那这里是为什么呢？如果 padding 空间也可以被利用，那 C2 为什么没有利用 C1 的 padding”？你自言自语着，思虑片刻，写下了两条思路：
- C3 并没有用到 C2 的 padding，因为在 C3 中，子类 C2 没有尾部 padding。C2 只有一个 char，它没有尾部对齐需求。C3 也只有一个 char，没有起始地址对齐需求。所以 C3 和 C2 紧挨在一起。C3 尾部的 padding 整个 C3 类对齐 C1 int* 的需求。虽然 C2 也没有头部 padding，但是 C1 有尾部 padding，所以 C2 不能紧挨着 C1； 
- 只有第一个子类的 padding 空间不能被利用，其它的都能； 

第二条思路中的“只有第一个子类”的说法看起来太特殊，而第一条思路虽然更复杂，但似乎更具有普适性，于是你选择先验证思路一。你重新设计了 C2，使它也必然产生尾部 padding，然后再看 C3 是否会利用其尾部 padding 的空间。你画出了 C1、C2 和 C3 的预期布局图：
![Padding-C1-C2-C3-Declaration-Layout-2-Expecting](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2_c3-declaration-layout-2-expecting.png) 

你试着运行下面的程序，但是结果却不是你预期的那样：  
``` c++
class C1 {
public:
    int* val1;
    char bit1;

    static void DisplayLayout(C1 *obj) {
        PRINT_START(C1);
        PRINT_OFFSET("C1", obj, &obj->val1);
        PRINT_OFFSET("C1", obj, &obj->bit1);
        PRINT_END();
    }
};

class C2: public C1{
public:
    int* val2;
    char bit2;

    static void DisplayLayout(C2 *obj) {
        PRINT_START(C2);
        PRINT_OFFSET("C1", obj, &obj->val1);
        PRINT_OFFSET("C1", obj, &obj->bit1);
        PRINT_OFFSET("C2", obj, &obj->val1);
        PRINT_OFFSET("C2", obj, &obj->bit2);
        PRINT_END();
    }
};

class C3: public C2{
public:
    char bit3;

    static void DisplayLayout(C3 *obj) {
        PRINT_START(C3);
        PRINT_OFFSET("C1", obj, &obj->val1);
        PRINT_OFFSET("C1", obj, &obj->bit1);
        PRINT_OFFSET("C2", obj, &obj->val1);
        PRINT_OFFSET("C2", obj, &obj->bit2);
        PRINT_OFFSET("C3", obj, &obj->bit3);
        PRINT_END();
    }
};

int main() {
    C1 c1;
    C1::DisplayLayout(&c1);

    C2 c2;
    C2::DisplayLayout(&c2);

    C3 c3;
    C3::DisplayLayout(&c3);
}
```

输出为：  
```
C1 Layout (Total Size 016):
SubObject           ::Attribute           : Offset(~Size) 0xAddress
C1                  ::&obj->val1          : 000000(~0008) 0x7ffc4300fdd0
C1                  ::&obj->bit1          : 000008(~0001) 0x7ffc4300fdd8
==========

C2 Layout (Total Size 032):
SubObject           ::Attribute           : Offset(~Size) 0xAddress
C1                  ::&obj->val1          : 000000(~0008) 0x7ffc4300fde0
C1                  ::&obj->bit1          : 000008(~0001) 0x7ffc4300fde8
C2                  ::&obj->val1          : 000000(~0008) 0x7ffc4300fde0
C2                  ::&obj->bit2          : 000024(~0001) 0x7ffc4300fdf8
==========

C3 Layout (Total Size 032):
SubObject           ::Attribute           : Offset(~Size) 0xAddress
C1                  ::&obj->val1          : 000000(~0008) 0x7ffc4300fe00
C1                  ::&obj->bit1          : 000008(~0001) 0x7ffc4300fe08
C2                  ::&obj->val1          : 000000(~0008) 0x7ffc4300fe00
C2                  ::&obj->bit2          : 000024(~0001) 0x7ffc4300fe18
C3                  ::&obj->bit3          : 000025(~0001) 0x7ffc4300fe19
==========
```

虽然 C1 和 C2 和预期的一致，但是 C3 中，C3 还是用到了 C2 的尾部 padding 空间：  
![Padding-C1-C2-C3-Layout-2](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2_c3-layout-2.png)  

你想着，既然思路一被证明不行，那就再试试思路二吧。思路二说只有第一个子类的 padding 空间不能被利用，其它的都能被利用。刚才的 C3 似乎也证明了这一点。但是刚才的 C3 只是单一继承的情况。如果 C3 是多重继承的呢？ 
- 第二继承链中的类会使用第一继承链中可用的 padding 空间吗？
- 只有第一个子类的 padding 空间不能被利用是针对所有继承链中的第一个子类而言的吗？还是说只是针对最左的继承链而言呢？

你设计了新的 C3 来检验上述第一点猜想，而事实证明，第二继承链中也会使用第一继承链中可用的 padding 空间：  
![Padding-C1-C2-C22-C3-Declaration-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2_c22_c3-declaration-layout.png)  

接下来你重新设计了 C2_2，使它继承自 C1_2，想用以观察 C2_2 是否会使用 C1_2 的 padding 空间：  
![Padding-C1-C2-C12-C22-C3-Declaration](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2_c12_c22_c3-declaration.png)  

结果发现，C2_2 并没有使用 C1_2 的尾部 padding:  
![Padding-C1-C2-C12-C22-C3-Layout](https://static-public-imhuwq.oss-cn-shenzhen.aliyuncs.com/writing/cpp-object-model/padding_c1_c2_c12_c22_c3-layout.png)  

也就是说，对于 Class 来说，任意继承链上的第一个子类(top 父类)尾部的 padding 是不能被其子类使用的。  

## 四、总结
事到如今，哪怕你读过 Inside the C++ Object Model，你也不得不承认在 C++ 对象数据对齐的问题上仍然有陷阱。在多级继承和多重继承中，数据对齐并不像 Lippman 所说的那样所有尾部 padding 都不能被使用，但也不是全部都能被使用。只能说，大部分都能用，但是任意继承链上的最顶级父类的尾部 padding 不能再被使用。  
而且，这很有可能是特定编译器的特定行为。所以说，如果我们遇到了这方面的困惑，**最好的方法是用代码来检验**。  
