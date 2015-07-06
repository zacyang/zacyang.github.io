---
layout: post
title:  "深入浅出宏"
date:   2015-06-30 10:29:11
categories: Clojure
---

###概述

###宏的历史及比较

###clojure中宏的实现

###编写一个有用的debug宏

###注意事项



### 代码块
{% highlight clojure %}
(defmacro some-macro []
    ~@alll
)

;;this is the coments
(defn church-number[])
(println "hello world")
{% endhighlight %}

{% highlight java %}
public class HelloWorld {
    public static void main(String args[]) {
      System.out.println("Hello World!");
    }
}
{% endhighlight %}

```clojure
(defn new-styl[] dosomething -> (do))
```
```python
print("hello, world")
```