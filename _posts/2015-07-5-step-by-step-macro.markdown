---
layout: post
title:  "深入浅出宏"
date:   2015-06-30 10:29:11
categories: Clojure
---
##为什么有这篇博客
> 接触clojure/lisp有一年多了，初衷只是为了把SICP上的题都做完。做到后面发现对这个语言族产生了很大的兴趣。于是把重点从clojure学习SICP变成了通过SICP更好得理解lisp。

> Clojure作为Lisp的一个JVM平台方言，对于没有Lisp经验的程序员来说学习曲线是非常陡的。我觉得其中有几个难点（按掌握有限顺序进行排序）

+ 学会把函数作为数据进行传递和处理
+ 通过递归分解和处理问题
+ filter accumulation 的通用模式的应用
+ 宏
+ 理解环境（$env) 在apply和eval中的作用


今天我们在这里来聊聊宏，因为有C语言的背景，所以内容不会仅限于Clojure中的宏。而在学习和应用宏的过程中遇到了很多问题，希望通过一篇博客把知识归纳总结出来。之所以是宏，是因为宏最难掌握切容易混淆其中的概念。
***

###宏的分类
##### 预编译宏 
<br>这里最典型的例子就是c语言。其中的宏定义如下，（当然比较好的作法是加上#ifdef).

```c
#define min(X, Y) \
    do { \
    ((X) < (Y) ? (X) : (Y)) \
    } while(0)\
```


 这段宏的意思是把找出最小（不限制类型），作为一个通用的过程以宏的形式提供出来。这样做的好处是，跳出了静态强类型语言对类型的限制，可以在更高的语义层次上提供抽象。即，对所有的事物比对大小。
 
 这段宏会在预编译期间把所有min(parameter1, paramerter2)的**文本内容**替换成对应的内容即
 
```c
 ((X) < (Y) ? (X) : (Y))
 
 //e.g   x = min(a, b);          ==>  x = ((a) < (b) ? (a) : (b));
 //      z = min(a + 28, *p);    ==>  z = ((a + 28) < (*p) ? (a + 28) : (*p));
```
 
 之后，完成预编译过程。并开始编译过程。
 
但是，即便这样短小的宏中我们也可以看到一些通用的问题，例如：

     *函数与宏，宏与宏的深度嵌套会导致语义过誉复杂。我们说过x y可以是任何东西，指针，结构体，函数。*

     *多重调用，如果传入一个函数f到min之中那么预编译后其结果就可能在运行过程中函数被多次调用*

     *自引用宏的问题，当一个宏交叉引用另一个宏时，出现定义递归，成为非法C代码* 


 
 
##### 抽象语法树宏      <br>由于lisp的方言其程序本身就是一棵[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)。这就省去了预编译这个过程的必要性，而事实也是几乎没有lisp的方言有预编译阶段。
当clojure文本被reader读入成为AST的时候，这个时候其中的宏就会在被调用时展开，替换对应树中的节点。

在我们开始具体理解clojure宏的细节和技巧之前，先让我们看看clojure中的宏是长什么样的，这里我们选取了很有用的语法糖宏"->",之所以选择这个宏是因为很多人对于lisp的这种操作符前置的语法很不习惯。
这个宏可以让我们以（貌似）面向对象的方式进行函数调用。

```clojure
;;选择序对的第3个元素，然后将其增加1
(def target-array [1 2 3])

(inc (second target-array))
;;=> 4

(-> target-array
    second
     inc)
     
```
这个宏可以让语义更加“可读”。*注意断句* ： 把第一个参数作为(第二个函数的第一个参数)进行求值，并以此类推。

 

###组装宏的螺丝和改刀

> 你需要知道的四种构建宏的工具

上面看到的这种宏其实给了我们一种可能性或者叫做能力。

+ 其一就是无限更改语言的面貌（语法、语法糖等）

比如下面这样就像另外一种语言的宏

```clojure
(my-macro
    somevar = somelist.get(1);
    println (somevar);
)

;;可以通过宏展开成

(let [somevar (-> somelist (get 1))]
    (println somevar)
)

```

而如何把这个宏做出来，（以及其他有点卵用的宏），我们将在接下来慢慢解开。

+ 另一种则是在更高抽象层次上提供组合功能（这一点
在c语言中的宏也是类似的，其中的异同会在后面提到）

##### 引用 Quote

我觉得有必要先解释一下引用，这个词的字面意思。（不要笑，我当时也不是很明白）

就拿scip中解释这个问题的经典例子吧。比如一个老师问学生“say your name”，回答往往是“tom”之类的；而如果问题是“say 'your name'”那么回答就是
“your name”。引用quote，这个词的意义就是原封不动的照搬字面意义。这个解释在clojure中也是成立的。

```clojure
(+ 1 2)
;;=> 3

(quote (+ 1 2))

;;=> （+ 1 2）

;;引用未绑定变量直接报错
none-exist-unbound-var
;;=>CompilerException java.lang.RuntimeException: 
;;Unable to resolve symbol: none-exist-unbound-var in this context, 
;;compiling:(/private/var/folders/gv/m43y5g9x1xdc9f2kc2p0kpz00000gn/T/form-init2497236881590386447.clj:1:5516)

;; quote 之后成为字面量
(quote none-exist-unbound-var)
;; => none-exist-unbound-var
```

下面这个例子是core中when的实现，其中就用了quote。

```clojure
(defmacro when
  "Evaluates test. If logical true, evaluates body in an implicit do."
  {:added "1.0"}
  [test & body]
  (list 'if test (cons 'do body)))
```

>quote的一个语法糖是' 

其实，在很多情况下，用quote就可以写出很多简单的宏了。其简单地思路就是通过list和quote把功能性函数和数据组装成未求值的表达式，然后在语法树上进行替换。
比如我们可以用这种方式做出并没有什么卵用的宏，只打印一个非nil的值：

```clojure
(defmacro but-no-nuan-use [x] (list 'if x (list 'println x)))

(but-no-nuan-use "hehe")
hehe
;=> "hehe"

(but-no-nuan-use nil)
;=> nil

```
可以看到把defmacro 换成 defn 其实也是一样工作的。但是宏和函数之间还有很多细小的差异，让我们在后面慢慢道来。

看了并没有什么卵用的宏之后，让我们看看有点点用的宏。这就离不开*语法引用*的帮助。

##### 语法引用 Syntax Quote 以及 反引用 Unquoting

首先，语法引用与引用的基本能力一样的。*但是*语法引用可以使用下面谈到的反引用。这是什么意思呢？

首先我们要知道，Lisp的程序就是一个抽象语法树，我们要得到一个值，就需要对树进行遍历求值。直到每一个叶进行eval，再求值上一层eval/apply，最终对更节点进行apply

这里，我们的宏会把其中的一些节点替换成宏定义的子语法树，而语法引用和引用可以办到的是，阻止eval对表达式进行求值，只返回字面量。

在前面的引用的例子中，我们的通用思路是通过list组装一个语法树字面量。 

```clojure
(list '<function> <parameter>)
```

等价于

```clojure
(<function> <parameter>)
```

但是，我们看到，如果有多个语句以及function需要调用，我们就需要为他们每一个都加上引用。这样会死人的。。。

看一下这个示例，只是简单把两个参数打印出来（实际工作中不会用这样没有卵用的宏，这里只是为了用最简单地方式来展示长语句宏）

```clojure
;;用list 加 quote的方式
(defmacro println-parameters
  "simplely println the 2 paramertes"
  [good bad]
  (list 'do
        (list 'println
              "Bad Item:"
              (list 'quote bad))
        (list 'println
              "Good item:"
              (list 'quote good))))
                            
```

这个示例中，我们完全无法接受对每个function都要进行quote，甚至，我们看到，对于quote本身，我们也需要进行quote来阻止其被求值。

让我们用语法引用做一个清新版的无卵用宏：

```clojure
(defmacro println-parameters
  "simplely println the 2 paramertes"
  [good bad]
  `(do
        (println
              "Bad Item:"
               ~bad)
        (println
              "Good item:"
               ~good)))
```

其中“~”是马上要提到的反引用。可以这样理解，在lisp的世界中，所有东西都有值，也会被求值，语法引用可以在这个世界中暂时建立一个不会被求值的（魔法）区域，
而在这个区域中，我们有一个咒语叫做反引用（unquaoting），他可以让语法引用（不求值）的魔力在一个小区域中消失，在这个小区域中Lisp世界的求值原则又可以应验。

对于这个例子来说，整个do开始的求值语法树都不会进行求值，只有good和bad被反引用了，那么求值器就会在当前$env中去搜索绑定变量good以及bad，那就是传入的参数。
然后把他们替换了。如果不加反引用会出现什么情况呢？由于无法在当前（和语法引用相同的）命名空间$env中找到绑定变量，那么宏会在运行时报错:"No Such var: user/bad"


语法引用还解决了引用在宏中的另一个问题，那就是命名空间问题。记得上面提到的报错吗？为什么回报错是user/bad?而不是说bad变量找不到？

这里就是语法引用与引用之间的另一个重大区别：

*引用是不会加入命名空间的*

```clojure
;引用，没有命名空间
'+
; => +

;语法引用，引入命名空间
`+
; => clojure.core/+
```

设想一下，如果不引入命名空间会出现什么情况。在宏的运行过程中，如果没有命名空间限定，那么eval被替换的语法树时，求值器会寻找最近的$env之中的绑定的计算过程。
（在lisp之中，一个计算过程就是一个lambda加上其环境env）。那么你完全无法控制宏的能力，比如我完全可以在引用宏的命名空间把“+”覆盖成一个打印函数。那么这会引入
太多的奇妙的问题。

##### 展开反引用

###开工 
>编写一个有用的debug宏

###宏的应用场景及注意事项

