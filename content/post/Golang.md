# Golang

- defer的执行顺序，能改变返回值吗

  defer的执行顺序和栈类似，属于后进先出

  defer在return之后执行，但是在函数退出之前，defer可以修改返回值。通过有名返回变量

  ```go
  func changeReturn() (i int) {
  	i = 32
  
  	defer func() {
  		fmt.Printf("defer \n")
  		i++
  	}()
  
  	return i
  }
  
  ```

  

- go 的tag用处

  - json序列化
  - gorm的表名映射
  - binding，validate 等gin web 框架需要的绑定和校验功能

- Golang的和java的区别？

  - golang是静态编译，编译完成后，即可在操作系统上运行。可以交叉编译打包不同os和arch的代码。java是动态编译，java中间存在一层jvm虚拟机，
  - java不支持多继承，golang通过struct的成员变量实现多继承
  - golang的goroutine只有2kb左右，而java的thread对应着内核的一个线程会达到2mb左右，资源开销不是一个级别。

- golang的反射说一下

  golang类似java的reflect提供了也提供了reflect的包，内部有相关的接口和方法使用。

  核心的两个type分别是 Type和Value

  golang提供了

  ```go
  	t := reflect.TypeOf(person)
  	v := reflect.ValueOf(person)
  ```

  通过反射获取field的相关信息以及tag等

  ```go
  func testReflect(person Person) {
  	t := reflect.TypeOf(person)
  	v := reflect.ValueOf(person)
  	for i := 0; i < t.NumField(); i++ {
  		field := t.Field(i)
  		value := v.Field(i).Interface().(string)
  		fmt.Printf("field %s tag %s value %s \n", field.Name, field.Tag, value)
  	}
  }
  ```

- golang的垃圾回收机制

- go slice是怎么扩容的？

  Go <= 1.17

  如果当前容量小于1024，则判断所需容量是否大于原来容量2倍，如果大于，当前容量加上所需容量；否则当前容量乘2。

  如果当前容量大于1024，则每次按照1.25倍速度递增容量，也就是每次加上cap/4。

- 有缓冲和无缓冲的channel区别

- make和new区别

  - make和new作用的变量类型不一样，make是在创建channel ，slice，map的时候使用的，new是string，int，以及自定义的struct使用
  - make返回的是引用类型，我对slice进行修改是直接修改这个引用的值，不会产生值的拷贝。new返回的是指针。相当于&struct{}

- channel死锁的场景

  - 当一个channel没有数据直接读取的时候会死锁，解决方案是通过select读取，设置default默认值

- 说说Context

  Context主要用在goroutine之前传参。

  Context有withValue等方法，可以将参数放在context中，进行传递

- go怎么排查内存泄露

  使用pprof

- map是安全的吗

  怎么保证线程安全

  使用mutex锁，通过channel进行交互。

- channel实现java里的coutdownlatch

  可以使用sync.WaitGroup

- 如何控制并发

  协程池，有相关的开源工具，类似ants

- golang的指针你说一下

  指针是一种数据类型，用于存储值的内存地址，通常在基本类型之前加*来表明是一个指针，new方法返回的就是指针。

  用法可以修改传值参数，因为在golang里只有值传递，所以默认情况下传的是值的副本。在函数中修改的话是无效的。

  出了map，slice，channel，这些属于引用。可以直接修改。

  所以如果要修改传参的值，则需要指针来处理。

