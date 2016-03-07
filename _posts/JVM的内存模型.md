#JVM的内存模型
 
##JVM内存结构
![内存结构][1]

  [1]: http://images0.cnblogs.com/blog/641601/201508/211701583165320.jpg
  
  其中方法区和java堆是线程共享区域，虚拟机栈 本地方法栈 程序计数器是线程隔离的区域。
  
##各内存结构的说明
### 1.方法区
方法区是各个线程共享的内存区域，用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据，又被人称为“永久代（永生代）”。在JDK1.7之前，字符串常量池是放在方法区，在1.7后，就将其从方法区移除放到java堆里。
**关联异常**：当方法区无法满足内存分配需求时，会抛出，OOM异常。

### 2. java堆
java堆是用于存放对象实例，几乎所有的对象都在这里分配内存，是在虚拟机启动时创建。同时也是垃圾收集器管理的主要区域，又被称为 GC堆，其里面分 新生代 和 老年代。
**设置堆内存参数** ： -Xms1m -Xmx1m -XX:+HeapDumpOnOutOfMemoryError（可以让虚拟机在出现内存溢出是dump出当前的内存堆转储快照）

### 3. 虚拟机栈
虚拟机栈是线程私有的，生命周期与线程一致，描述的是java方法执行的内存模型。每个方法在调用的时候都会创建一个栈帧，栈帧包含局部变量表、操作数栈、动态链接、方法出口等。其中局部变量表存放八种基本数据类型以及对象引用类型，其中，double和long会占用2个局部变量空间，其余的是一个局部变量空间。
**关联异常** ：当线程请求的栈深度大于虚拟机允许的最大栈深度，就会抛出SOF，比如调用递归。
如果虚拟机栈是可以动态扩展的，如果在扩展过程没有申请到内存空间，则会抛出OOM.

### 4. 本地方法栈
本地方法栈与虚拟机栈类似，只不过本地方法栈是虚拟机使用到的native方法服务。
异常跟虚拟机栈的一样。

### 5.程序计数器
线程私有的，作为当前线程所执行的字节码的行号指示器，比如通过程序计数器来获取下一条需要执行的字节码指令，如循环、跳转、异常处理等。

----
##代码测试关于内存的报错
###1.堆内存溢出

```java
/**
     * 堆内存设置:-Xms1m -Xmx1m
     * 在限制堆内存大小的时候,如果创建的对象申请不到内存,就会爆出内存溢出,并提示是 堆内存
     * java.lang.OutOfMemoryError: Java heap space
     */
    @Test
    public  void test() {
        ArrayList<Man2> mans = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            mans.add(new Man2());
        }
    }
```
###2.虚拟机栈内存溢出
当前线程申请的栈深度大于虚拟机允许的最大深度导致的栈内存溢出
```java
/**
 * 递归
 */
 public  void stackLeak() {
        stackLength++ ;
        stackLeak();
    }

    /**
     * 线程栈大小设置:-Xss160k
     * 栈帧:每个方法调用的时候,用栈帧存储局部变量表 操作数栈 动态链接 方法出口
     * 单线程下,如果是 栈帧过大 或者是 虚拟机栈容量太小,都会导致内存无法分配的时候,就会抛出 StackOverflowError
     */
    public static void main(String[] args) throws Throwable{
        JVMStackHeapSize jvmStackHeapSize = new JVMStackHeapSize();
        try {
            jvmStackHeapSize.stackLeak();
        } catch (Throwable e) {
            System.out.println(" stack length is " + jvmStackHeapSize.stackLength);
            throw e;
        }
    }
```
###3. 永生带：string 的intern方法
```java
    /**
     * intern : 如果字符串常量池中已经包含了一个等于此string对象的字符串,则返回代表翅中这个字符串的string对象;
     * 否则,将其加到常量池中.
     * 在jdk1.6及其以下版本,intern方法 会把首次遇到的字符串 放到永久代中,返回的也是永久代中这字符串的实例,
     * 所以下面2个都会是输出 false. 因为一个是有StringBuilder 创建的对象在 堆内存中,一个是永久代中.
     *
     * 在jdk1.7 的时候,去掉了永久代,调用intern方法时,不会再复制实例,而是在常量池 记录首次出现的实例引用,
     * 因此intern方法返回的是引用. 其中 "阿尼玛~" 是第一次出现这个字符串,所以 对比是同一个引用 因此为true
     * "java" 是之前就已经存在常量池的了,所以此次返回的引用与 stringBuilder 不是同一个引用,因而为false
     */
    @Test
    public void testIntern() {
        String itern1 = new StringBuilder("阿尼玛").append("~").toString();
        System.out.println(itern1 == itern1.intern());
        String itern2 = new StringBuilder("ja").append("va").toString();
        System.out.println(itern2 == itern2.intern());
    }

```