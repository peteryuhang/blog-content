---
title: Java 中的 Nested 和 Inner Class
author: yuhang
date: 2023-07-21 11:05:00
categories: [日志, 编程语言, Java]
tags: [编程语言, Java]
---

记得自己刚接触 Java 那会，有两个东西总是傻傻分不清，一个是 **Nested Class**，翻译过来叫做 **嵌套类**，另外一个是 **Inner Class**，翻译过来是 **内部类**。咦，嵌套不就是一个东西在另一个东西内部吗？其实这两个还是有很大区别的。

<br>

**在说一个技术时，我们最好结合一些问题来看，这样比较容易理解这个技术出现的原因，以及它要解决的问题**。在 Java 中，一切都是对象，类在 Java 中的地位不用我多说，我们熟知的是，类中可以定义变量，方法，那类中是否可以定义其他的类呢？答案是可以，毕竟类在 Java 中的地位是非常之高。这的确给类的使用带来诸多的灵活性，可问题也随之而来。我们知道，类是一个模块，为了方便对其管理，我们会对其中的内容进行权限的设定，比如按照惯例，类中的变量我们会设置为私有（private），方法会设置为公有（public）。如果我们允许在类中定义类的话，那么 **这两个类中的成员和方法该怎样相互访问？**

<br>

我们以实现 Java 中的 `ArrayList` 为例，为了清楚地表述，一开始我们先定义分开的两个类：

```java
public class ArrayList<AnyType> implements Iterable<AnyType> {
  private int theSize;
  private AnyType[] theItems;

  public java.util.Iterator<AnyType> iterator() {
    return new ArrayListIterator<AnyType>(this);
  }

  public AnyType[] getItems() {
    return theItems;
  }
}

class ArrayListIterator<AnyType> implements java.util.Iterator<AnyType> {
  private int current = 0;
  private ArrayList<AnyType> theList;
  public ArrayListIterator(ArrayList<AnyType> list) {
    thisList = list;
  }

  public boolean hasNext() {
    return current < theList.size();
  }

  public AnyType next() {
    return theList.getItems().theItems[current++];
  }
}
```

虽然说上面的代码可以正常的编译和运行，但是其中隐藏着诸多问题：

- `ArrayList` 和 `ArrayListIterator` 是分开的两个类，但是从从属关系来看，`ArrayListIterator` 从属于 `ArrayList` 并且 `ArrayListIterator` 仅仅会被 `ArrayList` 所使用，而当前的权限定义表明，`ArrayListIterator` 还可以被其他的类所使用，这其实传递了错误的信号
- `ArrayListIterator` 没法直接获取 `ArrayList` 的成员信息，构建 `ArrayListIterator` 还需要手动传入 `ArrayList` 对象，这使得类的某些实现变得复杂，比如 `theList.getItems().theItems[current++]` 
- 对于 `ArrayList` 的使用者来说，需要维护 `ArrayList` 和 `ArrayListIterator` 这层关系，单一情况还好，但有多个 `ArrayListIterator` 和多个 `ArrayList` 对象时，情况就会变得相当复杂

<br>

由此可见，有必要对其进行优化，优化方向是将上述的两个类合为一个类，这样 **对外方便管理，对内容易实现**。我们首先来看 **嵌套类**：

```java
public class ArrayList<AnyType> implements Iterable<AnyType> {
  private int theSize;
  private AnyType[] theItems;

  public java.util.Iterator<AnyType> iterator() {
    return new ArrayListIterator<AnyType>(this);
  }

  private static class ArrayListIterator<AnyType> implements java.util.Iterator<AnyType> {
    private int current = 0;
    private ArrayList<AnyType> theList;
    public ArrayListIterator(ArrayList<AnyType> list) {
      thisList = list;
    }

    public boolean hasNext() {
      return current < theList.size();
    }

    public AnyType next() {
      return theList.theItems[current++];
    }
  }
}
```

首先，对于嵌套类，我们必须使用 `static` 关键字指明它为嵌套类，简单解释就是，**嵌套类只是嵌套在类中，它是类的东西，和类所生成的对象无关**。相比于之前，有以下的提升：

- 我们可以把嵌套类的访问权限设定为私有，这样，`ArrayListIterator` 只能由 `ArrayList` 中的方法或成员访问。其对外不可见，降低了使用的复杂度
- 另外，`ArrayListIterator` 是 `ArrayList` 的成员，因而它能对 `ArrayList` 的成员进行访问，这多少降低了其功能实现的复杂度

但是，**嵌套类** 被用在此处还是有些问题：

- 从 `static` 关键字可以看出，嵌套类的访问是有局限性的，对其外部类的非静态成员和方法并没有直接的访问权限，还是只能通过外部类的对象来间接访问
- 从最终的功能实现上，我们希望每个 `ArrayList` 对象都有其对应的 `ArrayListIterator` 对象，这里的话只有一个 `ArrayListIterator`。当有多个 `ArrayList` 存在时，我们还是需要手动维护这层关系

<br>

最后，请出我们的 **内部类**：

```java
public class ArrayList<AnyType> implements Iterable<AnyType> {
  private int theSize;
  private AnyType[] theItems;

  public java.util.Iterator<AnyType> iterator() {
    return new ArrayListIterator<AnyType>();
  }

  /* 
  * inner class
  * - ArrayListIterator 是隐式范型, 可以套用外部类的范型，我们不需要指明
  */
  private class ArrayListIterator implements java.util.Iterator<AnyType> {
    private int current = 0;

    public boolean hasNext() {
      // ArrayList.this.size() 可以简单写成 size()
      return current < size();
    }

    public AnyType next() {
      // ArrayList.this.thisItems 可以简单写成 theItems
      return theItems[current++];
    }

    public void remove() {
      // 假设 ArrayList 中的 remove 会和 ArrayListIterator 中的 remove 产生冲突
      // 因此这里使用隐式引用 ArrayList.this.remove 来进行指代 ArrayList 中的 remove
      return ArrayList.this.remove(--current);
    }
  }
}
```

从上面的实现可以看出，现在的 `ArrayList` 和 `ArrayListIterator` 就是我们期望的，关于 **内部类**，有如下的内容是需要知道的：

- 内部类是隐式范型，它会沿用外部类的范型，我们无需指明
- 如果定义了内部类，编译器会生成一个外部类的隐式引用，假如外部类名为 `Outer`，那这个隐式引用就是 `Outer.this`。内部类的成员可以用这个隐式引用来访问外部类的成员，当然也可以直接访问外部类的成员。隐式引用主要是解决内部类和外部类的成员重名的问题
- 内部类所生成的对象无法独立于外部类的对象而存在

<br>

**内部类** 与 **嵌套类**，都是存在于其他类内部的成员，可分别看作是类的普通成员与静态成员。它们可以帮助我们更好地对模块进行封装，从而使用者无需知道它们与外部类的联系，降低了实现的耦合性。
