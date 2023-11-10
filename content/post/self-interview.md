---
title: "自我面试"
date: 2022-01-11T14:43:18+08:00
tags: ['面试']
---

- - golang 和 java区别

- 内存溢出和内存泄露区别

  - 怎么排查

- 用过什么消息队列？

  - 说下消息队列存在的意义

    异步，削峰，解耦

  - 如何解决消息按顺序消费

    首先，消息队列一大优势是异步，如果需要按顺序消费，则会变成同步，导致吞吐量下降，要确认是否一定需要顺序。

    - rabbitmq的几个概念

      Exchange： 消息交换机，它指定消息按什么规则，路由到哪个队列

      - Fanout`、`Direct` 和 `Topic
      - Fanout Exchange 会忽略 RoutingKey 的设置，直接将 Message 广播到所有绑定的 Queue 中。
      - Direct Exchange 是 RabbitMQ 默认的 Exchange，完全根据 RoutingKey 来路由消息。设置 Exchange 和 Queue 的 Binding 时需指定 RoutingKey（一般为 Queue Name），发消息时也指定一样的 RoutingKey，消息就会被路由到对应的Queue。
      - Topic Exchange 和 Direct Exchange 类似，也需要通过 RoutingKey 来路由消息，区别在于Direct Exchange 对 RoutingKey 是精确匹配，而 Topic Exchange 支持模糊匹配。分别支持`*`和`#`通配符，`*`表示匹配一个单词，`#`则表示匹配没有或者多个单词。

      Queue： 消息队列载体，每个消息都会被投入到一个或多个队列

      Binding： 绑定，它的作用就是把exchange和queue按照路由规则绑定起来

      Routing Key： 路由关键字，exchange根据这个关键字进行消息投递

  - 怎么保证不重复消费呢

    消费者做幂等就可以了，比如状态机，分布式锁，弄个数据库当记录表之类都可以。

  - 怎么保证不丢失消息呢

    - 生产端，开始channel的confirm模式，相当于是在发送完消息后，会等待broker确认，如果持久化成功了会回复ack，当然内部也是异步，springboot里面封装了callback的api
    - 消息队列自身保证，消息队列开启持久化功能，设置queue位durable
    - 消费者丢消息，把自动ack改为手动ack

  - 消息积压很大了怎么办

    - 紧急扩容，增加消费者实例，k8s这个时候增加副本数就行了

- 平时用什么db，mariadb

  优化过啥没

  - 不是专门dba，多数时间都是建索引，当然也得到百万级以上数据才会考虑

- java的重载返回值能不一样吗

  能，但方法名必须一致，参数必须不一样

- 面向对象

  **封装**的目的是隐藏事务内部的实现细节，以便提高安全性和简化编程。封装提供了合理的边界，避免外部调用者接触到内部的细节。我们在日常开发中，因为无意间暴露了细节导致的难缠bug太多了，比如在多线程环境暴露内部状态，导致的并发修改问题。从另外一个角度看，封装这种隐藏，也提供了简化的界面，避免太多无意义的细节浪费调用者的精力。

  **继承**是代码复用的基础机制，类似于我们对于马、白马、黑马的归纳总结。但要注意，继承可以看作是非常紧耦合的一种关系，父类代码修改，子类行为也会变动。在实践中，过度滥用继承，可能会起到反效果。

  **多态**，你可能立即会想到重写（override）和重载（overload）、向上转型。简单说，重写是父子类中相同名字和参数的方法，不同的实现；重载则是相同名字的方法，但是不同的参数，本质上这些方法签名是不一样的，为了更好说明，请参考下面的样例代码：

  ```java
      public int doSomething() {
          return 0;
      }
      // 输入参数不同，意味着方法签名不同，重载的体现
      public int doSomething(List<String> strs) {
          return 0;
      }
      // return类型不一样，编译不能通过
      public short doSomething() {
          return 0;
      }
  ```

  方法名称和参数一致，但是返回值不同，这种情况在Java代码中算是有效的重载吗？ 答案是不是的，编译都会出错的。

- string,stringbuilder,stringbuffer区别

  - string java.lang提供的类，并不是8个基本数据类型之一，8个基本数据类型是int,short,long,boolen,float,double,char,byte。

    string是不可变的，class和属性都是final修饰，保证多线程的安全。

  - stringbuilder是由于string是不可变，所以每次拼接等操作都是生成了新的string对象，会导致对象太多，占用内存。stringbuilder内部维护了byte或者char的数组，每次append都是新增数组尾部数据，并不会产生新对象。

    我们平时写的拼接字符串，反编译后，可以发现被jvm优化成用stringbuilder实现了

  - stringbuffer，内部使用了synchronized关键字来保证线程安全。

- java有哪些引用啊

  强引用，软引用，弱引用，虚引用

  - 平时我们生成对象进行赋值，这就是强引用，如果没有主动去除引用，即=null的操作，jvm垃圾回收器是不会进行回收的
  - 软引用，SoftReference，但jvm内存不够的时候，会进行回收这里面的对象，适合内存敏感的场景，例如如果查询数据不消耗很大资源，而需要缓存的时候，则可以用软引用，get不到，就重新塞入新对象
  - 弱引用，不管内存够不够都会被回收，就是维护了一种对象之间的映射关系，获取的时候，如果不get不到就生成新实例
  - 虚引用，不了解。

- 你谈谈一个类的加载过程吧

  我们写的.java文件会通过javac命令，编译成.class文件。jvm会将字节码文件，加载到元空间或者永久代里。

  这里会涉及到类加载机制，

  java提供了3个默认的类加载器BootStrapClassLoader，ExtClassLoader，AppClassLoader

  类加载器有几个特点

  - 双亲委派

    当一个类需要类加载器加载时，会先要求让父类加载器进行加载，依次往上。当然当前类加载器可以使用父类加载过的，防止多分字节码一样。自定义一个ClassLoader重写loadClass方法即可打破。Tomcat通过自定义一个webAppClassLoader来给每一个jar进行加载，达到 了间隔的效果。当然tomcat自身使用catalinaClassLoader进行加载的。类似的还有jenkins插件。

- 动态代理是什么？

  这实际上是一种设计模式，即代理模式。通过代理，我们可以在触发目标方法周围进行业务操作。典型的场景是AOP切面。

  实现代理的方式有JDK自带的代理，也有基于字节码修改的cglib工具。

  JDK自带的代理是通过java发射机制，java反射让我有能获取某个对象的定义，包括方法属性等，可以动态创建对象。

- hashmap聊一下

  - 如何扩容的？

    hashmap内部维护了几个变量capacity存储空间，threshold阈值，loadfactor负载因子

    当调用put等方法时，最终会触发resize()在resize内部会完成初始化以及扩容的操作

    第一次扩容即初始化的时候，默认容量为16，扩容门槛为16*0.75=12

    如果当前容量不为0，扩容新容量为旧容量的2倍，扩容门槛也为2倍。扩容阈值=当前容量*扩容因子

  - hashmap的结构说一下

    hashmap内部维护了一个数组，单位为node，也被称为bucket，当put key的时候，会key进行hash，得到hash值，然后与当前容量取余，得到数组下标。检查下标是否已经有元素了。如果有的话，默认情况下，会使用链表，链到结尾，默认情况下，链表长度如果达到8则进行树化，如果树退到6，则变成链表。

  - 为什么要变成红黑树

    在极差的情况下，hash一直冲突，则链表效率会太低，因为链表查询是O(n)的复杂度，而之所以用红黑树，因为二叉搜索树在最差的情况下，会退化到链表，而平衡二叉树，由于其平衡特性左右子树深度不能超过1，在最差情况下会一直平衡，导致平衡过于频繁，花费时间太多。而红黑树保证左右子树的深度不会超过2倍。和平衡二叉树比起来，在插入，删除频繁的场景下，平衡的时间很少。

  - 能key能null？value能null？

    key是可以为空的，且hash值为0，value可以为空。

  - hashmap的数组下标是如何计算的？

    hash值与table的size-1进行取余，为什么取余，循环数组下标啊。环形数组

    取余如果除数是2的N次方，则可以用或运算，公式hash & (size - 1)

  - 为什么计算hash要右移16位

    默认情况下hashmap的数组大小是1<<4=16。如果没有任何操作直接获取数据下标的话则是n&(16-1)=n&(15)=>n&(1111)，导致高位的数据，无法参与运算，导致得到的数组下标相同，影响效率。

    (h = key.hashCode()) ^ (h >>> 16)

    int是32

    将高16位与低16位进行异或，目的则是为了增加hash的散列程度，让数据分部更均匀。

- concurrentHashmap

  - 你先讲一下你知道java有哪些锁吧

    - synchronized ，java关键字，平时很常见的，过去性能较低，现在通过锁膨胀，性能优化很多。

      在运行时会有3种状态

      - 偏向锁，当同步代码一直被同一个线程访问，则这个线程会自动获取锁，降低获取锁的代价
      - 轻量级锁，当锁是偏向锁时，如果有另一个线程访问，锁会升级为轻量级锁，当前线程会自旋，不断尝试获取锁，
      - 重量级锁，当锁已经是轻量级锁的情况下，当自旋的线程已经自旋到一定程度的话，则会线程会进入阻塞状态，性能低

      自旋和阻塞区别？

      自旋只是当前线程在等待一会儿，并没有和内核状态切换的性能开销。如果同步代码块到代码过于简单，状态转换到时间有kennel比用户执行代码到时间还长。线程的挂起和恢复得不偿失。

      而阻塞，线程会进入阻塞状态，直到接受到信号被唤醒。

  - 说下线程的状态有哪些吧

    - New 新建状态
    - Runnable 运行状态，已经在jvm虚拟机中了，等待操作资源分配资源
    - Blocked 阻塞状态，等待一个monitor锁，平时我们使用synchronized的关键字，就会变成这个状态。或者被调用了Object.await()后，并且又被notify后也会进入这个状态。
    - Waiting状态，等待状态，Object.await()后，进入waiting状态。Thread.jon()方法，也会进入waiting状态，LockSupport.park()进入waiting状态

  - 线程能被终止吗

    - 可以通过在线程内部放一个标记，例如boolen字段，在内部while循环，从而从外部控制线程终止。

    - thread对象的interrupt方法，会给当前对象打上interrupted的标记，在线程内部可以判断这个标志。

      但是，如果当前线程是在阻塞状态，例如sleep，Object.await方法后。会抛出异常InterruptedException，当然我们可以捕捉这个异常然后退出线程。

      - interrupt()方法，给线程打上interrupted的标志
      - interrupted()方法，返回当前线程是否有interrupted的标志，如果有 则清除，下次会返回false
      - isInterrupted()方法，返回当前是否有interrupted的标志
      - stop，直接拔插头，会有问题

  - AQS是什么

    全程是AbstractQueueSynchronizer，抽象对象同步器

  - ReentrantLock是怎么实现重入锁，公平非公平锁，以及条件锁的

    - 重入锁，是指如果当前线程已经是获取锁的线程，则获取锁的时候不需要再做整个获取锁的逻辑，提高性能。除了ReentrantLock是可重入锁，synchronized关键字也是可重入锁，他是通过在对象头上的标记来判断当前线程是否是已经获取到锁了。但synchronized不是公平锁

      而ReentrantLock，ReetrantLock内部类Sync继承了AQS，而AQS内部维护了exclusiveOwnerThread变量，标记当前获取锁的线程，ReetrantLock会直接获取锁，并返回。

    - 公平锁，ReentrantLock利用了AQS提供了队列器，在尝试获取锁不成功后，会进入队列，然后阻塞。等待唤醒，一旦唤醒了，会检查是否该自己了，会继续进入循环，检查是否是需要获取锁的线程。而非公平锁则没有入队的操作，所以不公平，上来就直接尝试获取锁，即通过CAS更新锁状态。

  - volatile关键字的作用？

    java中的关键字，当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

    volatile只保证可见性，不保证原子性，比如 volatile修改的变量 i，针对i++操作，不保证每次结果都正确，因为i++操作是两步操作，相当于 i = i +1，先读取，再加1，这种情况volatile是无法保证的。

  - 写个java死锁的实例吧

    ```java
    import java.util.Date;
     
    public class LockTest {
       public static String obj1 = "obj1";
       public static String obj2 = "obj2";
       public static void main(String[] args) {
          LockA la = new LockA();
          new Thread(la).start();
          LockB lb = new LockB();
          new Thread(lb).start();
       }
    }
    class LockA implements Runnable{
       public void run() {
          try {
             System.out.println(new Date().toString() + " LockA 开始执行");
             while(true){
                synchronized (LockTest.obj1) {
                   System.out.println(new Date().toString() + " LockA 锁住 obj1");
                   Thread.sleep(3000); // 此处等待是给B能锁住机会
                   synchronized (LockTest.obj2) {
                      System.out.println(new Date().toString() + " LockA 锁住 obj2");
                      Thread.sleep(60 * 1000); // 为测试，占用了就不放
                   }
                }
             }
          } catch (Exception e) {
             e.printStackTrace();
          }
       }
    }
    class LockB implements Runnable{
       public void run() {
          try {
             System.out.println(new Date().toString() + " LockB 开始执行");
             while(true){
                synchronized (LockTest.obj2) {
                   System.out.println(new Date().toString() + " LockB 锁住 obj2");
                   Thread.sleep(3000); // 此处等待是给A能锁住机会
                   synchronized (LockTest.obj1) {
                      System.out.println(new Date().toString() + " LockB 锁住 obj1");
                      Thread.sleep(60 * 1000); // 为测试，占用了就不放
                   }
                }
             }
          } catch (Exception e) {
             e.printStackTrace();
          }
       }
    }
    ```

    

  - 如何实现锁安全的？

    分段锁

  - 线程安全的list用过什么

  - JVM的哪些地方会出现OOM的error

    首先JVM区域可以划分为如下

    - 程序计数器

      记录每个线程当前执行方法的地址

    - 虚拟机栈

      每执行一个方法，都会有一个栈帧入栈，栈帧存放着局部变量表，操作数栈等数据

    - 堆

      Java对象存放的地方，Jvm启动的时，可以通过Xmx参数控制大小，如果不控制，默认是物理内存的1/4。垃圾回收器还会对堆进行划分，例如新生代，老年代的划分。

    - 本地方法栈

      native方法的栈帧存放地方

    - 方法区

      用于存储所谓的元（Meta）数据，例如类结构信息，以及对应的运行时常量池、字段、方法代码等。现在是元空间，由于早期的Hotspot JVM实现，很多人习惯于将方法区称为永久代（Permanent Generation）。Oracle JDK 8中将永久代移除，同时增加了元数据区（Metaspace）。

      

    - 运行时常量区

      这是方法区的一部分。如果仔细分析过反编译的类文件结构，你能看到版本号、字段、方法、超类、接口等各种信息，还有一项信息就是常量池。Java的常量池可以存放各种常量信息，不管是编译期生成的各种字面量，还是需要在运行时决定的符号引用，所以它比一般语言的符号表存储的信息更加宽泛。

  - 内存溢出和内存逃逸是什么

    - 内存溢出，你要2个G大小，而内存只有1个G了

    -  内存泄漏：  (Memory Leak)

      强引用所指向的对象不会被回收，可能导致内存泄漏，虚拟机宁愿抛出OOM也不会去回收他指向的对象

      意思就是你用资源的时候为他开辟了一段空间，当你用完时忘记释放资源了，这时内存还被占用着，一次没关系，但是内存泄漏次数多了就会导致内存溢出

    - [内存](https://so.csdn.net/so/search?q=内存&spm=1001.2101.3001.7020)逃逸定义：

      在一段程序中，每一个函数都会有自己的内存区域存放自己的局部变量、返回地址等，这些内存会由编译器在栈中进

  - spring的bean是如何加载的呢

  - spring如何处理掉循环依赖的呢

    Spring通过三级缓存解决了循环依赖，其中一级缓存为单例池（`singletonObjects`）,二级缓存为早期曝光对象`earlySingletonObjects`，三级缓存为早期曝光对象工厂（`singletonFactories`）。当A、B两个类发生循环引用时，在A完成实例化后，就使用实例化后的对象去创建一个对象工厂，并添加到三级缓存中，如果A被AOP代理，那么通过这个工厂获取到的就是A代理后的对象，如果A没有被AOP代理，那么这个工厂获取到的就是A实例化的对象。当A进行属性注入时，会去创建B，同时B又依赖了A，所以创建B的同时又会去调用getBean(a)来获取需要的依赖，此时的getBean(a)会从缓存中获取，第一步，先获取到三级缓存中的工厂；第二步，调用对象工工厂的getObject方法来获取到对应的对象，得到这个对象后将其注入到B中。紧接着B会走完它的生命周期流程，包括初始化、后置处理器等。当B创建完后，会将B再注入到A中，此时A再完成它的整个生命周期。至此，循环依赖结束！

    面试官：”为什么要使用三级缓存呢？二级缓存能解决循环依赖吗？“

    答：如果要使用二级缓存解决循环依赖，意味着所有Bean在实例化后就要完成AOP代理，这样违背了Spring设计的原则，Spring在设计之初就是通过`AnnotationAwareAspectJAutoProxyCreator`这个后置处理器来在Bean生命周期的最后一步来完成AOP代理，而不是在实例化后就立马进行AOP代理。

  - 缓存一致性怎么解决？

    - 先删缓存再更新数据库，缓存删除完后，数据库还没有更新完成，此时下一个请求已经进来读到旧数据，缓存里又保留的是旧数据了。
      解决方法是，在第一个请求sleep一会，然后再删一次

    - 先更新数据库，再删缓存，缓存删失败了，依然是旧值

      先更新库，然后向消息队列里发送消息去更新缓存，利用消息队列的重试机制，保证缓存更新成功

    - 缓存设置超时时间就可以了

      注意要随机时间，否则会雪崩

  - mysql什么情况会不走索引

    - 没有遵循最左前缀匹配原则

  - mysql什么场景适合建索引，什么时候不适合

    - 数据分部均匀的适合建索引，重复数据太多不适合，例如性别不需要建索引。因为性别字段区分度很低，比如只有男女情况下，如果我要找数据，需要先进行第一次io，得到男女的所以对应的数据主键索引，在进行第二次io，而第一次男女结果后，数据并没有减少很多。所以没有必要。反而加重了一次io操作。速度反而会降低
    - 经常查询的where 后跟的字段，以及与其他表外键链接的字段，联合索引需要考察平时的语句是否是几个and一起，否则的话还是要分开建
    - 主键索引一定存在

    https://redspider.gitbook.io/concurrent/di-yi-pian-ji-chu-pian/5



