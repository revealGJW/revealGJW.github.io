---
title: 'Java8新特性:Lambda表达式和Stream'
date: 2019-02-12 17:00:46
categories:
	Java
---
## Lambda表达式
### 语法
一个lambda表达式就表示一个可被执行的代码块。与函数类似，lambda表达式具有`参数列表`以及`函数体`。不同之处在于lambda表达式没有函数名。
```
//参数列表
(Type1 param1, Type2 param2, ... , TypeN paramN) -> {
    //函数体
    statment1;
    statment2;
    //...;
    return statmentN;
}
```
这是一个最全的lambda表达式，其中的很多部分可以进行简化。
1. 参数类型
    - 绝大多数情况下编译器可以推断出lambda表达式的参数类型，所以上面表达式中的 `Type` 可以去掉。
2. 参数只有一个时，可以省略掉小括号。
3. lambda表达式只包含一条函数体时，可以省略掉花括号、return和语句结尾的分号。因此简化为：`param -> statement`

### 外部变量
lambda表达式可以访问外部变量，不过有一个限制是变量引用不可变。类似匿名内部类访问的外部变量也都是final修饰的。之所以这样是因为：
每个内部类的实例都隐藏了一个指向外部类实例的引用，只是没有显式地写出来而已。内部类访问外部类成员都是透过这个引用。之所以能有这个引用，是因为两者都是实例，有自己的内存空间；
而匿名内部类的外围环境函数只是一个函数，执行完之后，也就是匿名内部类诞生（初始化）完成的那一刻，它的生存周期就结束了。函数内部的局部变量（包括函数的参数）也就跟着被销毁了。所以产生出来的内部类根本无法像保留外部类的引用那样保留外围环境函数的引用。因此只能退而求其次，保留一份局部变量的拷贝值。

### 方法引用和构造器引用

#### 方法引用
方法引用主要用于简化lambda表达式的声明。语法格式有以下三种：
```
ObjectName::instanceMethod
ClassName::staticMethod
ClassName::instanceMethod
```
前两种方法相当于把lambda表达式的参数当成instanceMethod / staticMethod 的参数来调用，例如：`System.out::println` 等同于 `x->System.out.println(x)`
后一种等同于别把lambda表达式的第一个参数当成instanceMethod的目标对象，其他剩余参数当成该方法的参数，比如`String::toLowerCase` 等同于 `x->x.toLowerCase()`

#### 构造器引用
构造器引用语法如下：
```
Class::new
```
把lambda表达式的参数当成ClassName构造器的参数，例如：`BigDecimal::new` 等同于 `new BigDecimal(x)`

#### 总结
使用lambda表达式可以极大的简化代码，提高可读性。例如进行排序时不使用lambda表达式：
```
List<String> names = Lists.newArrayList("giao", "bolan", "mianj", "chuimandi");
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String o1, String o2) {
        return o1.compareTo(o2);
    }
});
```
使用lambda表达式：
```
Collections.sort(names, (o1, 02) -> o1.compareTo(o2));
```

## Stream
Stream是元素的集合，类似于iterrator；区别在于可以支持顺序和并行的对原Stream进行汇聚的操作。
例如：
```
List<Integer> nums = Lists.newArrayList(1,2,null,5,3,null);
nums.stream.filter(num -> num != null).count();
```
### 通用用法
![](http://img04.taobaocdn.com/imgextra/i4/90219132/T2ycFgXQ8XXXXXXXXX_!!90219132.jpg)

stream()创建了一个流对象，绿色部分将stream转换为另一个stream，生成新的stream，蓝色框进行汇聚。
因此使用stream的基本步骤有三个：
1. 创建Stream；
2. 转换Stream，每次转换原有Stream对象不改变，返回一个新的Stream对象（**可以有多次转换**）；
3. 对Stream进行聚合（Reduce）操作，获取想要的结果；

按照这个步骤进行归纳

### 创建Stream

创建stream有两个常用途径：
1. 通过Stream的静态工厂方法；
2. 通过Collection接口的默认方法，将Collection转换为Stream。。

使用静态方法：
1. of方法(可以传入单个或多个参数)

	```
    Stream<Integer> intgerStream = Stream.of(1, 2, 3);
    Stream<String> strStream = Stream.of("MT");
	```
2. generator方法
	生成一个无限长度的stream，元素通过给定的supplier生成
	```
    Stream.generate(new Supplier<Double> {
        @Override
        public Double get() {
            return Math.random();
        }
    }
    Stream.generate( () -> Math.random());
    Stream.generate(Math::random)
	```
	无限长度的stream使用的是懒加载，一般配合Stream的limit()方法使用。

3. iterate方法
	
	同样是生成无限长度的Stream，其元素声称是重复对给定的种子值(seed)调用用户指定函数生成的，可以理解其元素为：seed, f(seed), f(f(seed)),...
	```
    Stream.iterate(1, item -> item + 1).limit(10).forEach(System.out::println)
	```
    
Collection默认方法：
```
Collection.stream()
```
### 转换Stream
顾名思义，就是把一个Stream通过某些行为转换成一个新的Stream。下列是一些常用的转换方法：
1. **distinct** 
对Stream中包含的元素进行去重操作（依赖hashCode和equals方法），新生成的Stream中没有重复的元素。
![](http://img04.taobaocdn.com/imgextra/i4/90219132/T2K0lnXPRXXXXXXXXX_!!90219132.jpg)
2. **filter** 
对Stream中包含的元素使用给定的过滤函数进行过滤操作，新生成的Stream只包含复合条件的元素。
![](http://img03.taobaocdn.com/imgextra/i3/90219132/T2OxXnXPlXXXXXXXXX_!!90219132.jpg)
3. **map** 
对Stream中包含的元素使用给定的转换函数进行操作，新生成的Stream只包含转换生成的元素。
![](http://img03.taobaocdn.com/imgextra/i3/90219132/T2PQJnXOJXXXXXXXXX_!!90219132.jpg)
4. **flatMap** 
和map类似，不同的是其每个元素转换得到的是Stream对象，会把子Stream中的元素压缩到父集合中
![](http://img01.taobaocdn.com/imgextra/i1/90219132/T2mBXnXQhXXXXXXXXX_!!90219132.jpg)
5. **peek** 
生成一个包含原Stream所有元素的新Stream，同时会提供一个消费函数，新Stream每个元素被消费的时候都会执行给定消费函数；
![](http://img03.taobaocdn.com/imgextra/i3/90219132/T2DrFmXHtaXXXXXXXX_!!90219132.jpg)
6. **limit** 
对一个Stream进行截断操作，获取其前N个元素，如果原Stream中包含的元素个数小于N，那就获取其所有的元素；
![](http://img02.taobaocdn.com/imgextra/i2/90219132/T2QAXlXJBaXXXXXXXX_!!90219132.jpg)
7. **skip** 
返回一个丢弃原Stream的前N个元素后剩下元素组成的新Stream，如果原Stream中包含的元素个数小于N，那么返回空Stream；
![](http://img04.taobaocdn.com/imgextra/i4/90219132/T24A8mXUJXXXXXXXXX_!!90219132.jpg)

#### 使用示例
```
List<Integer> nums = Lists.newArrayList(1, 1, null, 3, 4, 6, null, 9, 10, 11, 12, 14);
System.out.println("sum is: " + 
nums.stream()
    .filter(num -> num != null)
    .distinct()
    .mapToInt(num -> num * 2)
    .peek(System.out::println)
    .skip(2)
    .limit(4)
    .sum());
```

#### 性能
Stream的写法看似对每个元素进行了多次操作，复杂度是O(n*N)，其实转换操作是lazy的，多个转换操作只会在汇聚操作的时候聚合起来，一次融合完成。可以理解为Stream中存在一个操作函数的集合，每次转换操作就是把这个转换函数放入集合中，汇聚时循环Stream，对每个元素执行所有的函数。

### 汇聚（Reduce）Stream
汇聚操作（也称为折叠）接受一个元素序列为输入，反复使用某个合并操作，把序列中的元素合并成一个汇总的结果。比如查找一个数字列表的总和或者最大值，或者把这些数字累积成一个List对象。
Stream接口有一些通用的汇聚操作，比如reduce()和collect()；也有一些特定用途的汇聚操作，比如sum(),max()和count()。注意：**sum方法不是所有的Stream对象都有的，只有IntStream、LongStream和DoubleStream是实例才有。**

#### 可变汇聚
把输入的元素们累积到一个可变的容器中，比如Collection或者StringBuilder。对应的方法时collect，它可以吧Stream中的元素收集到一个容器中。最通用的collect的定义：
```
<R> R collect(Supplier<R> supplier,
                  BiConsumer<R, ? super T> accumulator,
                  BiConsumer<R, R> combiner);
```
Supplier supplier是一个工厂函数，用来生成一个新的容器；
BiConsumer accumulator也是一个函数，用来把Stream中的元素添加到结果容器中；
BiConsumer combiner还是一个函数，用来把中间状态的多个结果容器合并成为一个（并发的时候会用到）。
举个栗子🌰：
```
List<Integer> nums = Lists.newArrayList(1,1,null,2,3,4,null,5,6,7,8,9,10);
    List<Integer> numsWithoutNull = nums.stream().filter(num -> num != null).
            collect(() -> new ArrayList<Integer>(),
                    (list, item) -> list.add(item),
                    (list1, list2) -> list1.addAll(list2));
```
第一个函数生成一个新的ArrayList实例；
第二个函数接受两个参数，第一个是前面生成的ArrayList对象，二个是stream中包含的元素，函数体就是把stream中的元素加入ArrayList对象中。第二个函数被反复调用直到原stream的元素被消费完毕；
第三个函数也是接受两个参数，这两个都是ArrayList类型的，函数体就是把第二个ArrayList全部加入到第一个中；
这样的方式看起来太过于复杂，因此collect还有另外一个版本，依赖Collector![http://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html]：
```
<R, A> R collect(Collector<? super T, A, R> collector);
```
Java8还提供了Collector的工具类，已经定义了一些静态工厂方法，例如Collectors.toCollection()收集到Collection中, Collectors.toList()收集到List中和Collectors.toSet()收集到Set中。使用Collectors的代码如下：
```
List<Integer> numsWithoutNull = nums.stream()
    .filter(num -> num != null)
    .collect(Collectors.toList());
```

#### 其他汇聚
- reduce方法：reduce方法非常的通用，后面介绍的count，sum等都可以使用其实现。reduce方法有三个override的方法，本文介绍两个最常用的，最后一个留给读者自己学习。先来看reduce方法的第一种形式，其方法定义如下：
```
Optional<T> reduce(BinaryOperator<T> accumulator);
```
接受一个BinaryOperator类型的参数，在使用的时候我们可以用lambda表达式来。
```
List<Integer> ints = Lists.newArrayList(1,2,3,4,5,6,7,8,9,10);
System.out.println("ints sum is:" + ints.stream().reduce((sum, item) -> sum + item).get());
```
可以看到reduce方法接受一个函数，这个函数有两个参数，第一个参数是上次函数执行的返回值（也称为中间结果），第二个参数是stream中的元素，这个函数把这两个值相加，得到的和会被赋值给下次执行这个函数的第一个参数。要注意的是：**第一次执行的时候第一个参数的值是Stream的第一个元素，第二个参数是Stream的第二个元素**。这个方法返回值类型是Optional，这是Java8防止出现NPE的一种可行方法，这里就简单的认为是一个容器，其中可能会包含0个或者1个对象。

reduce方法还有一个很常用的变种：
```
T reduce(T identity, BinaryOperator<T> accumulator);
```
这个定义上上面已经介绍过的基本一致，不同的是：它允许用户提供一个循环计算的初始值，如果Stream为空，就直接返回该值。而且这个方法不会返回Optional，因为其不会出现null值。下面直接给出例子，就不再做说明了。
```
List<Integer> ints = Lists.newArrayList(1,2,3,4,5,6,7,8,9,10);
System.out.println("ints sum is:" + ints.stream().reduce(0, (sum, item) -> sum + item));
```
– count方法：获取Stream中元素的个数。比较简单，这里就直接给出例子，不做解释了。
```
List<Integer> ints = Lists.newArrayList(1,2,3,4,5,6,7,8,9,10);
System.out.println("ints sum is:" + ints.stream().count());
```
- 搜索相关
    - allMatch：是不是Stream中的所有元素都满足给定的匹配条件
    - anyMatch：Stream中是否存在任何一个元素满足匹配条件
    - findFirst: 返回Stream中的第一个元素，如果Stream为空，返回空Optional
    - noneMatch：是不是Stream中的所有元素都不满足给定的匹配条件
    - max和min：使用给定的比较器（Operator），返回Stream中的最大|最小值

下面给出allMatch和max的例子：
```
List<Integer> ints = Lists.newArrayList(1,2,3,4,5,6,7,8,9,10);
System.out.println(ints.stream().allMatch(item -> item < 100));
ints.stream().max((o1, o2) -> o1.compareTo(o2)).ifPresent(System.out::println);
```
