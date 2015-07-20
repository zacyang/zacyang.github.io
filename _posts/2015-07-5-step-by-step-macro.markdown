---
layout: post
title:  "深入浅出宏"
date:   2015-06-30 10:29:11
categories: Clojure
---
##前言
> 接触clojure/lisp有一年多了，初衷只是为了把SICP上的题都做完。做到后面发现对这个语言族产生了很大的兴趣。于是把重点从clojure学习SICP变成了通过SICP更好得理解lisp。

> Clojure作为Lisp的一个JVM平台方言，对于没有Lisp经验的程序员来说学习曲线是非常陡的。我觉得其中有几个难点（按掌握有限顺序进行排序）

+ 学会把函数作为数据进行传递和处理
+ 通过递归分解和处理问题
+ filter accumulation 的通用模式的应用
+ 宏
+ 理解环境（$env) 在apply和eval中的作用


今天我们在这里来聊聊宏，因为有C语言的背景，所以内容不会仅限于Clojure中的宏。而在学习和应用宏的过程中遇到了很多问题，希望通过一篇博客把知识归纳总结出来。之所以是宏，是因为宏最难掌握切容易混淆其中的概念。
***

##宏的分类
### 预编译宏 
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


 
 
### 抽象语法树宏      <br>由于lisp的方言其程序本身就是一棵[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)。这就省去了预编译这个过程的必要性，而事实也是几乎没有lisp的方言有预编译阶段。
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

 

##组装宏的螺丝和改刀

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

### 引用 Quote

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

### 语法引用 Syntax Quote 以及 反引用 Unquoting

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

### 展开反引用 Unquote Splicing

前面我们提到了，在宏的开始一般是以语法引用+do开始的。这样做的目的是为了让未求值的序列，也就是lisp表达式能够顺序的运行。在开始介绍*展开反引用*之前，首先让我们来
看看下面的这个和你猜想会大大相反的例子。

```clojure
;;同样打印参数的函数
(defn fn-without-do [a b]
  (println "first line :" a)
  (println "second line :" b)  )
;;=>first line : :a
;;=>second line : :b
;;=>nil

;;用宏实现同样的功能
(defmacro macro-without-do [a b]
  `(println "first line :" ~a)
  `(println "second line :" ~b))
;;=>second line : :b
;;=>nil

```

为什么会出现这种差异，为什么宏里面第一行内容被忽略了？这开起来很简单地一个差异，其实关系到宏替换的本质问题，是文本替换，还是抽象语法树的替换。我们知道
clojure的宏是在抽象语法树中对一个节点用宏类容进行展开。那么既然是一个点的替换，那么宏也就只能eval出*一个clojure表达式*，这样才能形成一换一的状况，并
保证抽象语法树的完整性。**clojure的宏定义，只会返回最后一个表达式。**

所以，一般情况下我们需要在宏的内容之前加上do，来保证得到的是一个表达式。

那么下来，我们来看这个展开反引用是什么，以及为什么有它。考虑下面的情形，我需要把一个表达式比如(println 1)包装成(fn [] (println 1))的形式，我需要怎么做？
这里我们又会看到宏和函数的第二个不同，总之让我们用函数来试试：

```clojure
(defn proc->fn [proc]
  (fn [] (proc)))

(proc->fn (println 1))

1
;; => #<core$proc__GT_fn$fn__8009 sicp.ch2.core$proc__GT_fn$fn__8009@24b3941
```
呃，等等。为什么把(println 1)给我执行了啊？我需要的是把println 1作为被包装的函数体，然后返回给我这个函数啊。呵呵， 这里就是lisp求值的规律了。Lisp求值有两种顺序，
一种叫应用序，另一种叫做应用序；另一种叫做正则序。

**应用序**

即，在求值过程中，会在apply参数时，对参数进行首先求值，然后再替换函数体中对应的参数。

**正则序**

即，在求值时，首先把apply的参数原封不动的在函数体中进行替换，在上面的例子说就是把函数体中的proc 都替换成(println 1)。这也就是我们想要的效果。

但是，要看到如果使用正则序，应用程序的空间将会成倍上升。而大多数语言都使用的应用序求值。而上面这个proc->fn其实就是著名的延时求值的核心机制。通过函数来包裹
过程，并在需要的时候对函数求值拿出过程的值。

在这种情况下，我们就需要用到宏的几个能力，1.参数直接替换。2.body展开

```clojure
(defmacro proc->fn
  [proc]
  `(fn [] (~@proc)))

(let [delay-println (proc->fn (println 1))]
    (delay-println)
)
;=> 1
```

这里发生了什么事？首先，请出我们的老朋友

*macro-expand*

```clojure
(macroexpand `(proc->fn (println 1)))
;=> (fn* ([] (clojure.core/println 1)))
```
在这里，我们的(println 1)在运行时直接替换了proc位置上的值。那么为什么需要~@（展开反引用）呢？首先所有的宏最终的扩展形式都是

```clojure
(fn* ([] (body)))
```

其中body就是你的宏内容；而我们又知道每一个括号之中的表达式都会被直接求值。那么如果不加@（展开）只用~（反引用）的话，这个表达式的值就会是：

```clojure
(fn* ([] ((clojure.core/println 1))))
```

这里的问题就是，由于我们直接把(println 1)替换了proc，又知，proc==body,那么就会导致proc会被求值两次。在这个例子中，其结果就是在对 (delay-println)
进行求值时，会打印出1，然后抛出NullPointer。这个原因就是在求值过程中（内部括号）打印出了1，但是这个表达式的值是nil.而这个nil又会被外面的括号再次求值，所以导致
NullPointer.

这里~@展开反引用的作用就很清楚了，即， 把宏中的seq去掉第一次求值括号然后替换到后面的非绑定变量中去。


---------

####总结

*宏可以返回一个未被求值的clojure表达式。*

*你可以使用语法引用返回带命名空间的绑定变量，并且可以使用反引用来对引用中的变量进行求值*

*你可以使用list 以及原始引用（quote)组装未被求值的clojure表达式，但是会引入额外的开销即过多的quote.*

*宏的内容中只会返回一个表达式，所以如果要想在宏中返回多个form，那么请用do把这些表达式包装起来，如前例。*

*展开反引用(~@)可以用来阻止多表达式求值，达到字面替换的目的。在有多语句需要执行的场景下很有用*

