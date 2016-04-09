title: Java String 对 null 对象的容错处理
date: 2016-03-13 17:10:24
tags: [null, java, String]
categories: java
---

## 前言
最近在读《Thinking in Java》，看到这样一段话：
> Primitives that are fields in a class are automatically initialized to zero, as noted in the Everything Is an Object chapter. But the object references are initialized to null, and if you try to call methods for any of them, you’ll get an exception-a runtime error. Conveniently, you can still print a null reference without throwing an exception.
大意是：原生类型会被自动初始化为 0，但是对象引用会被初始化为 null，如果你尝试调用该对象的方法，就会抛出空指针异常。通常，你可以打印一个 null 对象而不会抛出异常。

第一句相信大家都会容易理解，这是类型初始化的基础知识，但是第二句就让我很疑惑：为什么打印一个 null 对象不会抛出异常？带着这个疑问，我开始了解惑之旅。下面我将详细阐述我解决这个问题的思路，并且深入 JDK 源码找到问题的答案。

## 解决问题的过程
可以发现，其实这个问题有几种情况，所以我们分类讨论各种情况，看最后能不能得到答案。

首先，我们把这个问题分解为三个小问题，逐一解决。

### 第一个问题
直接打印 null 的 String 对象，会得到什么结果？

    String s = null;
    System.out.print(s);

运行的结果是

    null

果然如书上说的没有抛出异常，而是打印了`null`。显然问题的线索在于`print`函数的源码中。我们找到`print`的源码：

    public void print(String s) {
        if (s == null) {
            s = "null";
        }
        write(s);
    }

看到源码才发现原来就只是加了一句判断而已，简单粗暴，可能你对 JDK 的简单实现有点失望了。放心，第一个问题只是开胃菜而已，大餐还在后面。

### 第二个问题
打印一个 null 的非 String 对象，例如说 Integer：

    Integer i = null;
    System.out.print(i);

运行的结果不出意料：

    null

我们再去看看`print`的源码：

    public void print(Object obj) {
        write(String.valueOf(obj));
    }

有点不一样的了，看来秘密藏在`valueOf`里面。

    public static String valueOf(Object obj) {
        return (obj == null) ? "null" : obj.toString();
    }

看到这里，我们终于发现了打印 null 对象不会抛出异常的秘密。`print`方法对 String 对象和非 String 对象分开进行处理。

1. **String 对象**：直接判断是否为 null，如果为 null 给 null 对象赋值为`"null"`。
2. **非 String 对象**：通过调用`String.valueOf`方法，如果是 null 对象，就返回`"null"`，否则调用对象的`toString`方法。

通过上面的处理，可以保证打印 null 对象不会出错。

到这里，本文就应该结束了。
什么？说好的大餐呢？上面还不够塞牙缝呢。
开玩笑啦。下面我们来探讨第三个问题。

### 第三个问题（隐藏的大餐）
null 对象与字符串拼接会得到什么结果？

    String s = null;
    s = s + "!";
    System.out.print(s);

结果可能你也猜到了：

    null!

为什么呢？跟踪代码运行可以发现，这回跟`print`没有什么关系。但是上面的代码就调用了`print`函数，不是它会是谁呢？`+`的嫌疑最大，但是`+`又不是函数，我们怎么看到它的源代码？这种情况，唯一的解释就是编译器动了手脚，天网恢恢，疏而不漏，找不到源代码，我们可以去看看编译器生成的字节码。

       L0
        LINENUMBER 27 L0
        ACONST_NULL
        ASTORE 1
       L1
        LINENUMBER 28 L1
        NEW java/lang/StringBuilder
        DUP
        INVOKESPECIAL java/lang/StringBuilder.<init> ()V
        ALOAD 1
        INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
        LDC "!"
        INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
        INVOKEVIRTUAL java/lang/StringBuilder.toString ()Ljava/lang/String;
        ASTORE 1
       L2
        LINENUMBER 29 L2
        GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
        ALOAD 1
        INVOKEVIRTUAL java/io/PrintStream.print (Ljava/lang/String;)V

看了上面的字节码是不是一头雾水？这里我们就要扯开话题，来侃侃`+`字符串拼接的原理了。

编译器对字符串相加会进行优化，首先实例化一个`StringBuilder`，然后把相加的字符串按顺序`append`，最后调用`toString`返回一个`String`对象。不信你们看看上面的字节码是不是出现了`StringBuilder`。详细的解释参考这篇文章 [Java细节：字符串的拼接][1]。

    String s = "a" + "b";
    //等价于
    StringBuilder sb = new StringBuilder();
    sb.append("a");
    sb.append("b");
    String s = sb.toString();

再回到我们的问题，现在我们知道秘密在`StringBuilder.append`函数的源码中。

    //针对 String 对象
    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }
    //针对非 String 对象
    public AbstractStringBuilder append(Object obj) {
        return append(String.valueOf(obj));
    }

    private AbstractStringBuilder appendNull() {
        int c = count;
        ensureCapacityInternal(c + 4);
        final char[] value = this.value;
        value[c++] = 'n';
        value[c++] = 'u';
        value[c++] = 'l';
        value[c++] = 'l';
        count = c;
        return this;
    }

现在我们恍然大悟，`append`函数如果判断对象为 null，就会调用`appendNull`，填充`"null"`。

## 总结
上面我们讨论了三个问题，由此引出 Java 中 String 对 null 对象的容错处理。上面的例子没有覆盖所有的处理情况，算是抛砖引玉。

如何让程序中的 null 对象在我们的控制之中，是我们编程的时候需要时刻注意的事情。

  [1]: http://droidyue.com/blog/2014/08/30/java-details-string-concatenation/