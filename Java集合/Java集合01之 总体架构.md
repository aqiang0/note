# Java集合01之 总体架构

**集合总体继承图**

![img](D:\MyDownloads\MyNotes\java集合\Java集合01之 总体架构.assets\08171028-a5e372741b18431591bb577b1e1c95e6.jpg)

从上图可知，Java集合分为两大派别，分别是**Collection**何**Map**

- Collection是一个接口，是高度抽象出来的集合，里面只是定义了该有什么方法，留给后代自己实现。

   Collection下面又有两大抽象接口，List和Set两大分支。

  - List存储的元素是有序的、可重复的。 List的实现类有LinkedList, ArrayList, Vector, Stack。

  - Set(注重独一无二的性质): 存储的元素是无序的、不可重复的。
         Set的实现类有HastSet和TreeSet。HashSet依赖于HashMap，它实际上是通过HashMap实现的；TreeSet依赖于TreeMap，它实际上是通过TreeMap实现的。

-  Map是一个映射接口，(用 Key 来搜索的专家): 使用键值对（kye-value）存储，类似于数学上的函数 y=f(x)，“x”代表 key，"y"代表 value，Key 是无序的、不可重复的，value 是无序的、可重复的，每个键最多映射到一个值。

    AbstractMap是个抽象类，它实现了Map接口中的大部分API。而HashMap，TreeMap，WeakHashMap都是继承于AbstractMap。 Hashtable虽然继承于Dictionary，但它实现了Map接口。

**Iterable接口：**正是由于每一个容器都有取出元素的功能。这些功能定义都一样，只不过实现的具体方式不同（因为每一个容器的数据结构不一样）所以对共性的取出功能进行了抽取，从而出现了Iterator接口。而每一个容器都在其内部对该接口进行了内部类的实现。也就是将取出方式的细节进行封装。Collection的父接口. 实现了Iterable的类就是可迭代的.并且支持增强for循环。

**Iterator：** iterator() 返回该集合的迭代器对象

这一系列文章转自：[如果天空不死](<https://home.cnblogs.com/u/skywang12345>) <https://home.cnblogs.com/u/skywang12345>

 