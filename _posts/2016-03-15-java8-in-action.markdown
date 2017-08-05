---
layout: post
title:  "java8 实践小结"
date:   2016-03-15 22:16
categories: java
permalink: /Priest/java8-in-action

---

**前言：** 厂里从去年已经推广java8并将线上所有应用都升级了java8，对厂里所有程序员来讲都是一种幸福，优雅的`Lambda`表达式以及`Stream`，让代码都变得更加优雅，但同时也提高了对程序猿的要求，因为一行简单的代码已经能够做很多工作了，某种角度上看，代码变得复杂了。经过几个月的踩坑、终于对java8 有个基本的了解，于是做个小结。

<h2>Lambda 访问范围</h2>

>能够访问局部对应的外部区域的局部final变量，以及成员变量和静态变量



```java
public class TestLambda {

  static int localVar1;

   int localVar2;
    
	@Test
	public void testVar() {
	    // sum作为 局部对应的外部区域的局部final变量,在编译时被隐式当做final变量处理
	    BigDecimal sum = BigDecimal.ZERO;
	    List<BigDecimal> numList = Arrays.asList(new BigDecimal(3), new BigDecimal(4));
	    /**numList.stream().forEach(n -> {sum = sum.add(n);}); 
	     * 编译出错,java: 从lambda 表达式引用的本地变量必须是最终变量或实际上的最终变量
	     */
	    System.out.println(sum.floatValue()); //0.0
	    BigDecimal numSum = numList.stream().reduce(sum, (n1, n2) -> n1.add(n2));
	    System.out.println(numSum.floatValue()); //7.0
	    //  对成员变量和静态变量 拥有读写权利
	    numList.stream().forEach(n -> {
	        localVar1 = 3;
	        localVar2 = 4;
	    });
	    System.out.println(localVar1 + " " + localVar2);  // 3 4
	}
}
```


<h2>java8 内置函数式接口</h2>

>包含:Predicate Function Consumer Supplier Optional Comparator
 其中常用的 Predicate Function Consumer 比较多
  

 `1. Predicate,boolean型函数,一般用于过滤`
 
```java
List<String> strList = Arrays.asList("123459", "1234", "2345345", "1230989");
// filterStr: [123459] , filter 的第二个参数就是 Predicate
List<String> filterStr = strList.stream().filter(s -> s.startsWith("1") && s.length() > 4 && s.endsWith("9") && s.length() < 7).collect(toList());
//一般条件很长的时候 代码可读性会比较低,所以 项目中的做法会这样
Predicate<String> stringPredicate = s -> s.startsWith("1") && s.length() > 4 && s.endsWith("9") && s.length() < 7;
List<String> filterStr2 = strList.stream().filter(stringPredicate).collect(toList());
//如果是特别长的情况,会是这样
Predicate<String> stringPredicate1 = s -> s.startsWith("1");
Predicate<String> stringPredicate2 = s -> s.length() > 4;
Predicate<String> stringPredicate3 = s -> s.endsWith("9");
Predicate<String> stringPredicate4 = s -> s.length() < 7;
List<String> filterStr3 = strList.stream().filter(stringPredicate1.and(stringPredicate2).and(stringPredicate3).and(stringPredicate4)).collect(toList());
```
  `2. Function 返回一个单一结果,实践上 一般用于 对象转换`
  
```java 
List<Integer> numbers = Arrays.asList(2);
Function<Integer, String> convertToStrFunc = n -> "00000" + n * n;
List<String> numStrs = numbers.stream().map(convertToStrFunc).collect(toList());
//function 有几个常用的 接口 默认方法实现, compose  andThen
Function<Integer, Integer> sumFunc1 = n -> n*n;
Function<Integer, Integer> sumFunc2 = n -> n*3;
//compose 先执行 参数sumFunc2 再执行 调用方sumFunc1,andThen反过来
// 2*3=6  6*6=36
List<Integer> r1 = numbers.stream().map(sumFunc1.compose(sumFunc2)).collect(toList());
// 2*2=4 4*3=12
List<Integer> r2 = numbers.stream().map(sumFunc1.andThen(sumFunc2)).collect(toList());
```

 `3.Consumer :  一个没有返回值的function, 一般用在forEach  有andThen 默认接口函数实现`

```java
List<Integer> testNums = Arrays.asList(1, 2, 3);
Consumer<Integer> numConsumer = n -> n = n*2;
testNums.stream().forEach(numConsumer);

//4.comparator :
Comparator<Integer> comparator = (t1, t2) ->t1.compareTo(t2);
testNums.sort(comparator);
Collections.reverse(testNums);
System.out.println(testNums);
```

<h2>Stream </h2>


>分顺序和分行 两种方式, 常用的有filter,map,reduce,flatMap Parallel Stream 并行方式来执行以 提升效率



 `1.filter,应用:找到集合第一个符合条件的对象并返回,filter().findFirst()组合可避免遍历完所有对象`
 

```java
List<Integer> testNums = Arrays.asList(4, 5, 6);
// 只输出一次 看来findFirst不仅仅有提升逼格的作用
Optional<Integer> matchNum = testNums.stream().filter(n -> {
    System.out.println(" current n is : " + n);
    return n % 2 == 0;
}).findFirst();
// 输出三次
Integer matchNum2 = testNums.stream().filter(n -> {
    System.out.println(" current e is : " + n);
    return n % 2 == 0;
}).collect(toList()).get(0);
```
 


` 2.map,映射对象到另外一个对象中去`


```java
List<String> numStrings = Arrays.asList("1", "2", "3");
int sum = numStrings.stream().mapToInt(Integer::valueOf).sum();
```



`3.reduce 对stream的元素进行增减操作`



```java      
        
BigDecimal initSum = BigDecimal.ONE;
List<BigDecimal> numList = Arrays.asList(new BigDecimal(3), new BigDecimal(4));
Optional<BigDecimal> finalSum = numList.stream().reduce((n1, n2) -> n1.add(n2)); // 7
BigDecimal resultSum = numList.stream().reduce(initSum, (n1, n2) -> n1.add(n2));// 8
```     
 


`4. flatMap 用于获取 所有集合对象的子集合`



```java     
        
Order order01 = new Order(1, "01", Arrays.asList(new OrderDetail(1), new OrderDetail(1)));
Order order02 = new Order(2, "02", Arrays.asList(new OrderDetail(3), new OrderDetail(4)));
Order order03 = new Order(1, "03", Arrays.asList(new OrderDetail(5), new OrderDetail(6)));
List<Order> orders = Arrays.asList(order01, order02, order03);
List<OrderDetail> orderDetailList = orders.stream().flatMap(o -> o.getOrderDetails().stream()).collect(toList());

// parallelStream 并行流,使用了 java7 的fork/join 进行处理, 提升了速度
List<String> numStrings2 = Arrays.asList("1", "2", "3");
int sum2 = numStrings.parallelStream().mapToInt(Integer::valueOf).sum();
    
```




<h2>Collectors </h2>

>常用的toList toSet groupingBy toMap


`1. toMap 将集合转map`


```java

Order order01 = new Order(1, "01", Arrays.asList(new OrderDetail(1), new OrderDetail(1)));
Order order02 = new Order(2, "02", Arrays.asList(new OrderDetail(3), new OrderDetail(4)));
Order order03 = new Order(1, "03", Arrays.asList(new OrderDetail(5), new OrderDetail(6)));
List<Order> orders = Arrays.asList(order01, order02, order03);
//map(keyFunc, valueFunc) ,在 key重复的时候 运行会报错
Map<Integer, Order> orderMap = orders.stream().collect(Collectors.toMap(o -> o.getId(), o -> o));
//map(keyFunction,valueFunc,mergeFunction)  mergeFunction 指定在key重复的时候使用哪一个value
Map<Integer, Order> orderMap2 = orders.stream().collect(Collectors.toMap(o -> o.getId(), o -> o, (k1, k2) -> k1));
orderMap.entrySet().forEach(e -> System.out.println(e.getKey()));
```


`2.groupingBy 聚合操作`


```java

Map<Integer, List<Order>> orderGroupMap = orders.stream().collect(Collectors.groupingBy(Order::getId));

Map<Integer, Integer> groupNumMap = orders.stream().collect(Collectors.groupingBy(Order::getId, Collectors.summingInt(p -> 1)));

```



这是一些基本的常用的用法，在使用过程中，曾经遇到过一个这样的疑惑，类似这种 `list.filter(filterFunc).forEach(changeElementValue)` 这样操作能改变原来的`list`的值吗？于是测试了下：


```java
@Test
public void testFilerChangeObject() {
    List<String> nums = Arrays.asList("1", "5");
    nums = nums.stream().filter(e -> Integer.parseInt(e) > 2).collect(toList());
    nums.forEach(e -> e = e + "aaa");
    System.out.println(nums);  // [1,5]
 
    List<Order> orders = Arrays.asList(new Order(1, "001", null), new Order(2, "002", null));
    orders.stream().filter(e -> e.getId()>1).forEach(o -> o.setNo("111111"));
    System.out.println(orders);
}
```


实验结果，基本数据类型 和  String 都不能修改生效，但是普通的对象修改能生效。原因：基本类型和String作为参数传入方法时， 无论该参数在方法内怎样被改变，外部的变量原型总是不变的，因为改动的是值的一个拷贝，所以没有改变。普通对象的修改的是对象的引用，所以能生效。详情可以参考[这里](http://freej.blog.51cto.com/235241/168676).