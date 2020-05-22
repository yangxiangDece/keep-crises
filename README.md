# Java基础

## 集合

### HashMap

- 1.7
  
  - 数组 + 链表、扩容时头插法
  
- 1.8
  
  - 数组 + 链表 + 红黑树、扩容时采用 尾插法
  - 当链表的深度达到8的时候，也就是默认阈值，就会自动扩容把链表转成红黑树的数据结构来把时间复杂度从O（n）变成O（logN）提高了效率
  
- JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法，那么他们为什么要这样做呢？
  
  - 因为JDK1.7是用单链表进行的纵向延伸，当采用头插法时会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题.
  - 尾插法还会保持元素原本的顺序
  
- 为什么HashMap的容量总是2的n次幂？

  - 关键代码p = tab[i = (n - 1) & hash]，n是一定是2的幂次方，2的幂次方换算成二进制，高位一定是1，然后再减去1，就变成除了低位第一位为0，其他全部为1，所以当hash的二进制和（n-1）的二进制进行位与运算的时候，hash的二进制任何一位变成0或1，那么最终得到的值是不同的，这样扩大了数组散列性
  - 2的幂次方：方便位运算、数据均匀分布
  - 如果设置的不是2的幂次方，HashMap会计算出与该数最接近的数字

- 扩容机制
  - ```java
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    ```
  
  - 扩容大小为原数组的2倍
  
  - 为什么在JDK1.8中进行对HashMap优化的时候，把链表转化为红黑树的阈值是8？
  
    - 数学 —— 泊松分布
    - 桶的长度超过8的概率非常非常小。所以作者应该是根据概率统计而选择了8作为阀值
  
  - 1.7是先扩容再进行插入
  
    - 当你发现你插入的桶是不是为空，如果不为空说明存在值就发生了hash冲突，那么就必须得扩容，但是如果不发生Hash冲突的话，说明当前桶是空的（后面并没有挂有链表），那就等到下一次发生Hash冲突的时候在进行扩容，如果以后都没有发生hash冲突产生，那么就不会进行扩容了，减少了一次无用扩容，也减少了内存的使用
    
  - load_factor 负载因子越大
  
    - 优点：空间利用率高
    - 缺点：Hash冲突概率加大、链表变长、查找效率变低
  
  - load_factor 负载因子越小
  
    - 优点：Hash冲突概率小、链表短、查找效率高
    - 缺点：空间利用率低、频繁扩容耗费性能
  
- 并发情况下的问题
  - 插入的元素可能被覆盖
  - put的时候链表可能形成环形数据结构，会形成死循环 主要原因还是线程不安全
  
- 为什么 HashMap 中 String、Integer 这样的包装类适合作为 key 键？

  - String、Integer 等是 final 类型 ，不可变，保证了 key 的不可更改性，就保证了 Hash值的不可更改性，不会出现放入时和取出时hash不同的情况
  - 内部已经重写了eqauls()、hashcode() ，不容易出现hash值的计算错误

- 为什么不直接采用经过`hashCode（）`处理的哈希码 作为 存储数组`table`的下标位置？

  - 容易出现哈希码与数组大小范围不匹配的情况，即计算出来的哈希码可能不在数组大小范围内，从而导致无法匹配存储位置
  - HashMap的解决方案：哈希码 & (数组长度 - 1)

- 为什么采用 哈希码 **与运算(&)** （数组长度-1） 计算数组下标？

  - 要想让算出来的hash值在数组范围内，就只能取余，即：h % length；但是取余效率低，这个是操作系统决定的，所以采用位运算 &

- hash计算规则

- ```java
  static final int hash(Object key) {
      int h;
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
  ```

- put 源码

- ```java
  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                     boolean evict) {
          Node<K,V>[] tab; Node<K,V> p; int n, i;
          if ((tab = table) == null || (n = tab.length) == 0)
              n = (tab = resize()).length;
          if ((p = tab[i = (n - 1) & hash]) == null)
              tab[i] = newNode(hash, key, value, null);
          else {
              Node<K,V> e; K k;
              if (p.hash == hash &&
                  ((k = p.key) == key || (key != null && key.equals(k))))
                  e = p;
              else if (p instanceof TreeNode)
                  e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
              else {
                  for (int binCount = 0; ; ++binCount) {
                      if ((e = p.next) == null) {
                          p.next = newNode(hash, key, value, null);
                          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                              treeifyBin(tab, hash);
                          break;
                      }
                      if (e.hash == hash &&
                          ((k = e.key) == key || (key != null && key.equals(k))))
                          break;
                      p = e;
                  }
              }
              if (e != null) { // existing mapping for key
                  V oldValue = e.value;
                  if (!onlyIfAbsent || oldValue == null)
                      e.value = value;
                  afterNodeAccess(e);
                  return oldValue;
              }
          }
          ++modCount;
          if (++size > threshold)
              resize();
          afterNodeInsertion(evict);
          return null;
      }
  ```

- get 源码

- ```java
  final Node<K,V> getNode(int hash, Object key) {
          Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
          if ((tab = table) != null && (n = tab.length) > 0 &&
              (first = tab[(n - 1) & hash]) != null) {
              if (first.hash == hash && // always check first node
                  ((k = first.key) == key || (key != null && key.equals(k))))
                  return first;
              if ((e = first.next) != null) {
                  if (first instanceof TreeNode)
                      return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                  do {
                      if (e.hash == hash &&
                          ((k = e.key) == key || (key != null && key.equals(k))))
                          return e;
                  } while ((e = e.next) != null);
              }
          }
          return null;
      }
  ```

### ConcurrentHashMap 

- 1.7
  
  - 数组 + 链表
  - segment 分段锁
    - Segment数组的意义就是将一个大的table分割成多个小的table来进行加锁，而每一个Segment元素存储的是多个HashEntry数组+链表，同时Segment继承了ReentrantLock
    - 定位一个元素过程需要进行两次Hash操作，第一次Hash定位Segment，第二次Hash定位元素所在数组位置
    - 坏处：定位Hash过程较长
    - 好处：写操作的时候可以只对元素所在的Segment进行加锁即可，不会影响到其他的Segment，这样，在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作
  
- 

- 1.8
  - 数组 + 链表 + 红黑树
  
  - CAS + synchronized — cas失败自旋保证成功 — 再失败就用sync保证
  
  - 在ConcurrentHashMap中通过一个Node<K,V>[]数组来保存添加到map中的键值对，而在同一个数组位置是通过链表和红黑树的形式来保存的。但是这个数组只有在第一次添加元素的时候才会初始化，否则只是初始化一个ConcurrentHashMap对象的话，只是设定了一个sizeCtl变量，这个变量用来判断对象的一些状态和是否需要扩容
  
  - 第一次添加元素的时候，默认初期长度为16，当往map中继续添加元素的时候，通过hash值跟数组长度取与来决定放在数组的哪个位置，如果出现放在同一个位置的时候，优先以链表的形式存放，在同一个位置的个数又达到了8个以上，如果数组的长度还小于64的时候，则会扩容数组。如果数组的长度大于等于64了的话，在会将该节点的链表转换成树
  
  - 在扩容完成之后，如果某个节点的是树，同时现在该节点的个数又小于等于6个了，则会将该树转为链表
  
  - put源码
  
    - ```java
      /*
       * 当添加一对键值对的时候，首先会去判断保存这些键值对的数组是不是初始化了，
       * 如果没有的话就初始化数组，然后通过计算hash值来确定放在数组的哪个位置
       * 如果这个位置为空则直接添加，如果不为空的话，则取出这个节点来
       * 如果取出来的节点的hash值是MOVED(-1)的话，则表示当前正在对这个数组进行扩容，复制到新的数组，则当前线程也去帮助复制
       * 最后一种情况就是，如果这个节点，不为空，也不在扩容，则通过synchronized来加锁，进行添加操作
       * 然后判断当前取出的节点位置存放的是链表还是树
       * 如果是链表的话，则遍历整个链表，直到取出来的节点的key来个要放的key进行比较，如果key相等，并且key的hash值也相等的话，则说明是同一个key，则覆盖掉value，否则的话则添加到链表的末尾
       * 如果是树的话，则调用putTreeVal方法把这个元素添加到树中去
       * 最后在添加完成之后，会判断在该节点处共有多少个节点（注意是添加前的个数），如果达到8个以上了的话，
       * 则调用treeifyBin方法来尝试将处的链表转为树，或者扩容数组
       */
      final V putVal(K key, V value, boolean onlyIfAbsent) {
              if (key == null || value == null) throw new NullPointerException();//K,V都不能为空，否则的话跑出异常
              int hash = spread(key.hashCode());//取得key的hash值
              int binCount = 0;//用来计算在这个节点总共有多少个元素，用来控制扩容或者转移为树
              for (Node<K,V>[] tab = table;;) {
                  Node<K,V> f; int n, i, fh;
                  if (tab == null || (n = tab.length) == 0)    
                      tab = initTable();//第一次put的时候table没有初始化，则初始化table
                  else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {//通过哈希计算出一个表中的位置因为n是数组的长度，所以(n-1)&hash肯定不会出现数组越界
                      if (casTabAt(tab, i, null,//如果这个位置没有元素的话，则通过cas的方式尝试添加，注意这个时候是没有加锁的
                                   new Node<K,V>(hash, key, value, null)))//创建一个Node添加到数组中区，null表示的是下一个节点为空
                          break;                   // no lock when adding to empty bin
                  }
                  /*
                   * 如果检测到某个节点的hash值是MOVED，则表示正在进行数组扩张的数据复制阶段，
                   * 则当前线程也会参与去复制，通过允许多线程复制的功能，一次来减少数组的复制所带来的性能损失
                   */
                  else if ((fh = f.hash) == MOVED)    
                      tab = helpTransfer(tab, f);
                  else {
                      /*
                       * 如果在这个位置有元素的话，就采用synchronized的方式加锁，
                       *     如果是链表的话(hash大于0)，就对这个链表的所有元素进行遍历，
                       *         如果找到了key和key的hash值都一样的节点，则把它的值替换到
                       *         如果没找到的话，则添加在链表的最后面
                       *  否则，是树的话，则调用putTreeVal方法添加到树中去
                       *  
                       *  在添加完之后，会对该节点上关联的的数目进行判断，
                       *  如果在8个以上的话，则会调用treeifyBin方法，来尝试转化为树，或者是扩容
                       */
                      V oldVal = null;
                      synchronized (f) {
                          if (tabAt(tab, i) == f) {//再次取出要存储的位置的元素，跟前面取出来的比较
                              if (fh >= 0) {//取出来的元素的hash值大于0，当转换为树之后，hash值为-2
                                  binCount = 1;            
                                  for (Node<K,V> e = f;; ++binCount) {//遍历这个链表
                                      K ek;
                                      if (e.hash == hash &&//要存的元素的hash，key跟要存储的位置的节点的相同的时候，替换掉该节点的value即可
                                          ((ek = e.key) == key ||
                                           (ek != null && key.equals(ek)))) {
                                          oldVal = e.val;
                                          if (!onlyIfAbsent)//当使用putIfAbsent的时候，只有在这个key没有设置值得时候才设置
                                              e.val = value;
                                          break;
                                      }
                                      Node<K,V> pred = e;
                                      if ((e = e.next) == null) {//如果不是同样的hash，同样的key的时候，则判断该节点的下一个节点是否为空，
                                          pred.next = new Node<K,V>(hash, key,//为空的话把这个要加入的节点设置为当前节点的下一个节点
                                                                    value, null);
                                          break;
                                      }
                                  }
                              }
                              else if (f instanceof TreeBin) {//表示已经转化成红黑树类型了
                                  Node<K,V> p;
                                  binCount = 2;
                                  if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,//调用putTreeVal方法，将该元素添加到树中去
                                                                 value)) != null) {
                                      oldVal = p.val;
                                      if (!onlyIfAbsent)
                                          p.val = value;
                                  }
                              }
                          }
                      }
                      if (binCount != 0) {
                          if (binCount >= TREEIFY_THRESHOLD)//当在同一个节点的数目达到8个的时候，则扩张数组或将给节点的数据转为tree
                              treeifyBin(tab, i);    
                          if (oldVal != null)
                              return oldVal;
                          break;
                      }
                  }
              }
              addCount(1L, binCount);//计数
              return null;
          }
      ```
  
  - get源码
  
    - ```java
      /*
       * 相比put方法，get就很单纯了，支持并发操作，
       * 当key为null的时候回抛出NullPointerException的异常
       * get操作通过首先计算key的hash值来确定该元素放在数组的哪个位置
       * 然后遍历该位置的所有节点
       * 如果不存在的话返回null
       */
      public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
          if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
              return e.val;
          }
          else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
          while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
              return e.val;
          }
        }
        return null;
      }
      ```
  
  - 扩容机制
  
    - ```java
      /**
       * 把数组中的节点复制到新的数组的相同位置，或者移动到扩张部分的相同位置
       * 在这里首先会计算一个步长，表示一个线程处理的数组长度，用来控制对CPU的使用，
       * 每个CPU最少处理16个长度的数组元素,也就是说，如果一个数组的长度只有16，那只有一个线程会对其进行扩容的复制移动操作
       * 扩容的时候会一直遍历，直到复制完所有节点，没处理一个节点的时候会在链表的头部设置一个fwd节点，这样其他线程就会跳过他，
       * 复制后在新数组中的链表不是绝对的反序的
       */
      ```
  
    - 所以引起数组扩容的情况如下：
  
      - 只有在往map中添加元素的时候，在某一个节点的数目已经超过了8个，同时数组的长度又小于64的时候，才会触发数组的扩容
      - 当数组中元素达到了sizeCtl的数量的时候，则会调用transfer方法来进行扩容
  
    - 那么在扩容的时候，可以不可以对数组进行读写操作呢？
  
      - 事实上是可以的。当在进行数组扩容的时候，如果当前节点还没有被处理（也就是说还没有设置为fwd节点），那就可以进行设置操作
      - 如果该节点已经被处理了，则当前线程也会加入到扩容的操作中去
  
    - 那么，多个线程又是如何同步处理的呢？
  
      - 在ConcurrentHashMap中，同步处理主要是通过Synchronized和unsafe两种方式来完成的
      - 在取得sizeCtl、某个位置的Node的时候，使用的都是unsafe的CAS方法，来达到并发安全的目的
      - 当需要在某个位置设置节点的时候，则会通过Synchronized的同步机制来锁定该位置的节点
      - 在数组扩容的时候，则通过处理的步长和fwd节点来达到并发安全的目的，通过设置hash值为MOVED
      - 当把某个位置的节点复制到扩张后的table的时候，也通过Synchronized的同步机制来保证线程安全

### Arrays.asList

- asList 得到的知识一个Arrays 的内部类，一个原来数组的视图List，如果对他进行增删操作会报错
- 用 ArrayList 的构造器可以将其转变为真正的 ArrayList

### 集合 fail-fast 策略

- 判断modCount 是否与 exceptedModCount 相等，若不相等，则表示出现并发有其他线程修改了集合；若相等则未出现并发情况
- 抛出 ConcurrentModificationExcetion 异常

### 如何在遍历的同时删除ArrayList中的元素

- 直接使用普通for循环进行删除；

  - 不能再foreach中进行删除，但是可以使用普通的for循环，因为普通的for雄厚没有用到Iterator的遍历，所以就不会进行 fail-fast的检验

  - ```java
    List<String> userNames = new ArrayList<String>() {{
      add("Hollis");
      add("hollis");
      add("HollisChuang");
      add("H");
    }};
    
    for (int i = 0; i < 1; i++) {
      if (userNames.get(i).equals("Hollis")) {
        userNames.remove(i);
      }
    }
    System.out.println(userNames);
    ```

  - 这种方式存在一个问题，那就是remove操作会改变List中元素的下标，可能存在漏删的问题

- 直接使用Iterator进行删除

  - ```java
    List<String> userNames = new ArrayList<String>() {{
      add("Hollis");
      add("hollis");
      add("HollisChuang");
      add("H");
    }};
    
    Iterator iterator = userNames.iterator();
    
    while (iterator.hasNext()) {
      if (iterator.next().equals("Hollis")) {
        iterator.remove();
      }
    }
    System.out.println(userNames);
    
    ```

  - 使用Iterator提供的remove方法，就可以修稿到expectedModCount的值，那么就不会抛出异常了8中

- 使用Java8中提供的filter过滤生成新的集合

### CopyOnWriteArrayList

- CopyOnWriteArrayList相当于线程安全的ArrayList，CopyOnWriteArrayList使用了一种叫写时复制的方法，当有新元素add到CopyOnWriteArrayList时，先从原有的数组中拷贝一份出来，然后在新的数组做写操作，写完之后，再将原来的数组引用指向到新数组
- 这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器
- 注意：CopyOnWriteArrayList的整个add操作都是在锁的保护下进行的。也就是说add方法是线程安全的。

## 枚举

- 枚举单例模式

  - ```java
    public enum Singleton {
      INSTANCE;
      public void doSomething() {
         System.out.println("doSomething");
      }
      // 调用方式
      public static void main(String[] args) {
          Singleton.INSTANCE.doSomething();
      }
    }
    ```

  - 通过将定义好的枚举[反编译](http://www.hollischuang.com/archives/58)，其实枚举在经过`javac`的编译之后，会被转换成形如`public final class T extends Enum`的定义，枚举中的各个枚举项是通过 static 来定义的

  - 这个类是 final 类型的，不能被继承

    - ```java
      public enum T {
          SPRING,SUMMER,AUTUMN,WINTER;
      }
      ```

    - ```java
      public final class T extends Enum
      {
          //省略部分内容
          public static final T SPRING;
          public static final T SUMMER;
          public static final T AUTUMN;
          public static final T WINTER;
          private static final T ENUM$VALUES[];
          static
          {
              SPRING = new T("SPRING", 0);
              SUMMER = new T("SUMMER", 1);
              AUTUMN = new T("AUTUMN", 2);
              WINTER = new T("WINTER", 3);
              ENUM$VALUES = (new T[] {
                  SPRING, SUMMER, AUTUMN, WINTER
              });
          }
      }
      ```

  - 当一个Java类第一次被真正使用到的时候静态资源被初始化、Java类的加载和初始化过程都是线程安全的（因为虚拟机在加载枚举的类的时候，会使用ClassLoader的loadClass方法，而这个方法使用同步代码块保证了线程安全）所以，创建一个enum类型是线程安全的

## IO

#### BIO、NIO、AIO

- BIO，同步阻塞式IO，简单理解：一个线程处理一个连接，发起和处理IO请求都是同步的
- NIO，同步非阻塞IO，简单理解：一个线程处理多个连接，发起IO请求是非阻塞的但处理IO请求是同步的
- AIO，异步非阻塞IO，简单理解：一个有效请求一个线程，发起和处理IO请求都是异步的
- BIO里用户最关心“我要读”，NIO里用户最关心"我可以读了"，在AIO模型里用户更需要关注的是“读完了”
- NIO一个重要的特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作是同步的（消耗CPU但性能非常高）

#### 阻塞IO 和 非阻塞IO

- 这两个概念是程序级别的。主要描述的是程序请求操作系统IO操作后，如果IO资源没有准备好，那么程序该如何处理的问题：前者等待；后者继续执行（并且使用线程一直轮询，直到有IO资源准备好了）

#### 同步IO 和 非同步IO

- 这两个概念是操作系统级别的。主要描述的是操作系统在收到程序请求IO操作后，如果IO资源没有准备好，该如何相应程序的问题：
  - 前者不响应，直到IO资源准备好以后
  - 后者返回一个标记（好让程序和自己知道以后的数据往哪里通知），当IO资源准备好以后，再用事件机制返回给程序

#### Linux种IO模型

- 阻塞式IO模型

  - 当用户线程发出IO请求之后，内核会去查看数据是否就绪，如果没有就绪就会等待数据就绪，而用户线程就会处于阻塞状态，用户线程交出CPU。当数据就绪之后，内核会将数据拷贝到用户线程，并返回结果给用户线程，用户线程才解除block状态

  - ```java
    data = socket.read();
    ```

  - 如果数据没有就绪，就会一直阻塞在read方法

- 非阻塞IO模型

  - 当用户线程发起一个read操作后，并不需要等待，而是马上就得到了一个结果。如果结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦内核中的数据准备好了，并且又再次收到了用户线程的请求，那么它马上就将数据拷贝到了用户线程，然后返回，所以事实上，在非阻塞IO模型中，用户线程需要不断地询问内核数据是否就绪，也就说非阻塞IO不会交出CPU，而会一直占用CPU

  - ```java
    while(true) {
    	data = socket.read();
    	if (data != error) {
    		// 处理数据
    		break;
    	}
    }
    ```

  - 但是对于非阻塞IO就有一个非常严重的问题，在while循环中需要不断地去询问内核数据是否就绪，这样会导致CPU占用率非常高，因此一般情况下很少使用while循环这种方式来读取数据

- IO 复用模型

  - 在多路复用IO模型中，会有一个线程不断去轮询多个socket的状态，只有当socket真正有读写事件时，才真正调用实际的IO读写操作。因为在多路复用IO模型中，只需要使用一个线程就可以管理多个socket，系统不需要建立新的进程或者线程，也不必维护这些线程和进程，并且只有在真正有socket读写事件进行时，才会使用IO资源，所以它大大减少了资源占用
  - Linux支持IO多路复用的系统调用有select、poll、epoll，这些都是内核级别的，但select、poll、epoll本质上都是同步I/O，先是block住等待就绪的socket，再是block住将数据从内核拷贝到用户内存

- 异步IO模型

## 动态代理

#### 实现方式

- JDK动态代理：java.lang.reflect 包中的Proxy类和InvocationHandler接口提供了生成动态代理类的能力
- Cglib动态代理：Cglib (Code Generation Library )是一个第三方代码生成类库，运行时在内存中动态生成一个子类对象从而实现对目标对象功能的扩展
- JDK动态代理和Cglib动态代理的区别 
  - 使用动态代理的对象必须实现一个或多个接口
  - 使用cglib代理的对象则无需实现接口，达到代理类无侵入

## 语法糖

#### 糖块一、switch 支持 String与枚举

- ```java
  public class switchDemoString {
      public static void main(String[] args) {
          String str = "world";
          switch (str) {
          case "hello":
              System.out.println("hello");
              break;
          case "world":
              System.out.println("world");
              break;
          default:
              break;
          }
      }
  }
  ```

- 反编译后

- ```java
  public class switchDemoString
  {
      public switchDemoString()
      {
      }
      public static void main(String args[])
      {
          String str = "world";
          String s;
          switch((s = str).hashCode())
          {
          default:
              break;
          case 99162322:
              if(s.equals("hello"))
                  System.out.println("hello");
              break;
          case 113318802:
              if(s.equals("world"))
                  System.out.println("world");
              break;
          }
      }
  }
  ```

- 原来字符串的switch是通过equals()和hashCode()方法来实现的

#### 糖块二、泛型

- 泛型擦除，所有类型都是Object类型

- 对于Java虚拟机来说，他根本不认识`Map map`这样的语法。需要在编译阶段通过类型擦除的方式进行解语法糖

- ```java
  Map<String, String> map = new HashMap<String, String>();
  map.put("name", "hollis");
  map.put("wechat", "Hollis");
  map.put("blog", "www.hollischuang.com");
  ```

- 解语法糖后

- ```java
  Map map = new HashMap();
  map.put("name", "hollis");
  map.put("wechat", "Hollis");
  map.put("blog", "www.hollischuang.com");
  ```

- 虚拟机中没有泛型，只有普通类和普通方法，所有泛型类的类型参数在编译时都会被擦除，泛型类并没有自己独有的Class类对象。比如并不存在List<String>.class或是List<Integer>.class，而只有List.class

#### 糖块三、自动装箱与拆箱

- 自动装箱

  - ```java
     public static void main(String[] args) {
        int i = 10;
        Integer n = i;
    }
    ```

  - 反编译后

  - ```java
    public static void main(String args[])
    {
        int i = 10;
        Integer n = Integer.valueOf(i);
    }
    ```

- 自动拆箱

  - ```java
    public static void main(String[] args) {
    
        Integer i = 10;
        int n = i;
    }
    ```

  - 反编译后

  - ```java
    public static void main(String args[])
    {
        Integer i = Integer.valueOf(10);
        int n = i.intValue();
    }
    ```

  - 在装箱的时候自动调用的是`Integer`的`valueOf(int)`方法。而在拆箱的时候自动调用的是`Integer`的`intValue`方法

#### 糖块四、方法变长参数

- ```java
  public static void main(String[] args)
      {
          print("Holis", "公众号:Hollis", "博客：www.hollischuang.com", "QQ：907607222");
      }
  
  public static void print(String... strs)
  {
      for (int i = 0; i < strs.length; i++)
      {
          System.out.println(strs[i]);
      }
  }
  ```

- 反编译后

- ```java
   public static void main(String args[])
  {
      print(new String[] {
          "Holis", "\u516C\u4F17\u53F7:Hollis", "\u535A\u5BA2\uFF1Awww.hollischuang.com", "QQ\uFF1A907607222"
      });
  }
  
  public static transient void print(String strs[])
  {
      for(int i = 0; i < strs.length; i++)
          System.out.println(strs[i]);
  
  }
  ```

- 从反编译后代码可以看出，可变参数在被使用的时候，他首先会创建一个数组，数组的长度就是调用该方法是传递的实参的个数，然后再把参数值全部放到这个数组当中，然后再把这个数组作为参数传递到被调用的方法中

#### 糖块五、枚举

- ```java
  public enum t {
      SPRING,SUMMER;
  }
  ```

- 反编译后

- ```java
  public final class T extends Enum
  {
      private T(String s, int i)
      {
          super(s, i);
      }
      public static T[] values()
      {
          T at[];
          int i;
          T at1[];
          System.arraycopy(at = ENUM$VALUES, 0, at1 = new T[i = at.length], 0, i);
          return at1;
      }
  
      public static T valueOf(String s)
      {
          return (T)Enum.valueOf(demo/T, s);
      }
  
      public static final T SPRING;
      public static final T SUMMER;
      private static final T ENUM$VALUES[];
      static
      {
          SPRING = new T("SPRING", 0);
          SUMMER = new T("SUMMER", 1);
          ENUM$VALUES = (new T[] {
              SPRING, SUMMER
          });
      }
  }
  ```

- 通过反编译后代码我们可以看到，public final class T extends Enum，说明，该类是继承了Enum类的，同时final关键字告诉我们，这个类也是不能被继承的。当我们使用enmu来定义一个枚举类型的时候，编译器会自动帮我们创建一个final类型的类继承Enum类，所以枚举类型不能被继承

#### 糖块六、内部类

- 内部类之所以也是语法糖，是因为它仅仅是一个编译时的概念，outer.java里面定义了一个内部类inner，一旦编译成功，就会生成两个完全不同的.class文件了，分别是outer.class和outer$inner.class。所以内部类的名字完全可以和它的外部类名字相同

#### 糖块七、条件编译

- 所以，Java语法的条件编译，是通过判断条件为常量的if语句实现的。其原理也是Java语言的语法糖。根据if判断条件的真假，编译器直接把分支为false的代码块消除。通过该方式实现的条件编译，必须在方法体内实现，而无法在整个Java类的结构或者类的属性上进行条件编译，这与C/C++的条件编译相比，确实更有局限性。在Java语言设计之初并没有引入条件编译的功能，虽有局限，但是总比没有更强

#### 糖块八、数值字面量

- ```java
  public class Test {
      public static void main(String... args) {
          int i = 10_000;
          System.out.println(i);
      }
  }
  ```

- 反编译后

- ```java
  public class Test
  {
    public static void main(String[] args)
    {
      int i = 10000;
      System.out.println(i);
    }
  }
  ```

- 反编译后就是把_删除了。也就是说 编译器并不认识在数字字面量中的_，需要在编译阶段把他去掉

#### 糖块九、for-each

- for-each的实现原理其实就是使用了普通的for循环和迭代器

#### 糖块十、try-with-resource

- ```java
  public static void main(String... args) {
      try (BufferedReader br = new BufferedReader(new FileReader("d:\\ hollischuang.xml"))) {
          String line;
          while ((line = br.readLine()) != null) {
              System.out.println(line);
          }
      } catch (IOException e) {
          // handle exception
      }
  }
  ```

- 其实背后的原理也很简单，那些我们没有做的关闭资源的操作，编译器都帮我们做了。所以，再次印证了，语法糖的作用就是方便程序员的使用，但最终还是要转成编译器认识的语言

#### 糖块十一、Lambda表达式

#### 泛型擦除的坑

- ```java
      public static void method(List<String> list) {
          System.out.println("invoke method(List<String> list)");
      }
  
      public static void method(List<Integer> list) {
          System.out.println("invoke method(List<Integer> list)");
      }
  }
  ```

- 上面这段代码，有两个重载的函数，因为他们的参数类型不同，一个是List另一个是List ，但是，这段代码是编译通不过的。因为我们前面讲过，参数List和List编译之后都被擦除了，变成了一样的原生类型List，擦除动作导致这两个方法的特征签名变得一模一样

# Java并发编程

## 位运算

- 位异或运算（^）：运算规则是：两个数转为二进制，然后从高位开始比较，如果相同则为0，不相同则为1
- 位与运算符（&）：两个数都转为二进制，然后从高位开始比较，如果两个数都为1则为1，否则为0
- 位或运算符（|）：两个数都转为二进制，然后从高位开始比较，两个数只要有一个为1则为1，否则就为0
- 位非运算符（~）：如果位为0，结果是1，如果位为1，结果是0

## sleep()

- 让一个线程进入阻塞状态，但是不会释放锁，当时时间结束后，就会立即拿到锁，进入就绪状态

## join()

- 让主线程等待子线程执行完毕以后，再继续执行主线程的代码。join内部是调用的wait方法来实现的

## Java对象头

- 第一部分：存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。官方称为 Mark Word
- 第二部分：类型指针，对象执行它的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例
- 如果对象是一个Java数组，那在对象头中还必须用一块记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中却无法确定数组的大小

## synchronized

- 内部是通过monitorenter、monitorexter指令和monitor对象来实现的；
- ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象，在Hotspot源码中)_
- owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1
- 若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒
- 若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)；由此看来，monitor对象存在于每个Java对象的对象头中(存储的指针的指向)
- synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因
- notify/notifyAll和wait方法，在使用这3个方法时，必须在synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常，这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象。同时notify/notifyAll方法调用后，并不会马上释放监视器锁，而是在相应的synchronized(){}/synchronized方法执行结束后才自动释放锁，这是交给jvm处理的
- synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间
- 在Java 6之后Java官方对从JVM层面对synchronized较大优化
- synchronized是可重入锁，在一个线程调用synchronized方法的同时在其方法体内部调用该对象另一个synchronized方法，也就是说一个线程得到一个对象锁后再次请求该对象锁，是允许的，这就是synchronized的可重入性
- 出现异常时会释放锁

## 锁优化

### 偏向锁

- 如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，偏向锁的释放不需要做任何事情，这也就意味着加过偏向锁的MarkValue会一直保留偏向锁的状态，因此即便同一个线程持续不断地加锁解锁，也是没有开销的，偏向锁只有在不同线程请求锁是才会升级为轻量级锁，偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁
- 偏向锁默认开启，可以通过JVM参数关闭偏向锁-XX:-UseBiaseLocking=false，那么默认进入轻量级锁状态
- 原理：当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁，如果测试成功，表示线程已经获得了锁，如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程

### 轻量级锁

- 使用CAS来竞争锁，两条或两条以上的线程竞争同一个锁，则轻量级锁会膨胀成重量级锁
- 线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，则自旋获取锁，当自旋获取锁仍然失败时，表示存在其他线程竞争锁(两条或两条以上的线程竞争同一个锁)，则轻量级锁会膨胀成重量级锁

### 自旋锁

- 线程获取锁失败，就过一会再去获取，比如让线程去执行一个无意义的循环，循环结束后再去重新竞争锁，如果竞争不到继续循环，循环过程中线程会一直处于running状态，但是基于JVM的线程调度，会出让时间片，所以其他线程依旧有申请锁和释放锁的机会。自旋需要合理的配置

### 重量级锁

### 锁消除/锁粗化

- 将一些细小的锁，其实不会出现安全问题，但是很多细小的锁，会导致锁竞争，效率低下，可以将多个小锁扩

  展到一个大锁，这样可以减少锁的竞争。这里的大指的的范围

### 锁池

- 假设线程A已经拥有了某个对象(注意:不是类)的锁，而其它的线程想要调用这个对象的某个synchronized方法(或者synchronized块)，由于这些线程在进入对象的synchronized方法之前必须先获得该对象的锁的拥有权，但是该对象的锁目前正被线程A拥有，所以这些线程就进入了该对象的锁池中

### 等待池

- 假设一个线程A调用了某个对象的wait()方法，线程A就会释放该对象的锁后，进入到了该对象的等待池中，等待池中的线程不会去竞争该对象的锁；当有线程调用了对象的 notifyAll()方法（唤醒所有 wait 线程）或 notify()方法（只随机唤醒一个 wait 线程），被唤醒的的线程便会进入该对象的锁池中，锁池中的线程会去竞争该对象锁

## volatile

- 保证变量的可见性、防止指令重排序，内部采用内存屏障来实现的
- 如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。volatile并不能保证线程安全问题，只保证可见性，不保证数据一致
- 可见性
  - 嗅探机制 强制失效 处理器嗅探总线
- 有序性
  - 禁止指令重排序 lock前缀指令 内存屏障

## happens-before

- 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作
- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁
- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读
- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C
- 注意：两个操作之间具有happens-before关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前

## as-if-serial

- 不管怎么重排序（编译器和处理器为了提高效率），单线程 程序的执行结果不能被改变。
- 编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变程序结果
- 如果操作之间不存在依赖关系，那么编译器和处理器会重排序

## 线程间怎么通信？

- wait/notify机制、共享变量 synchronized 或者 lock 同步机制等
- volatile
- CountDonwLatch
- CyclicBarrier

## ThreadLocal 用来解决什么问题？

- 解决线程数据隔离

## 如何尽可能提高多线程并发性能？

- 使用ThreadLocal
- 减少线程切换
- 使用读写锁 copyonwrite 等机制 这些方面回答

## 读写锁适用于什么场景？

- 读写锁适合并发多、写并发少的场景
- copyonwrite

## 如何实现一个生产者与消费者模型？

- 锁
- 信号量
- 线程通信
- 阻塞队列

## AQS (AbstractQueuedSynchronizer) 队列同步器

- 同步器的设计是基于模板方法模式的，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法
- 子类推荐被定义为自定义同步装置的内部类，同步器拥有三个成员变量：sync队列的头结点head、sync队列的尾节点tail和状态state。对于锁的获取，请求形成节点，将其挂载在尾部，而锁资源的转移（释放再获取）是从头部开始向后进行

## BlockingQueue

## CopyOnWrite

## ConcurrentSkipListMap

## ConcurrentLikedQueue

## Semaphore

## CountDownLatch

## CyclicBarrier

## LockSupport

## CompletableFuture

## Fork/Join框架

## Atomic*

## Unsafe

## 线程池

- 工作原理
  - 如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）
  - 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue
  - 如果无法将任务加入BlockingQueue（队列已满），判断当前运行的线程数是否超过maxmumPoolSize，如果未超出则创建新的线程来处理任务；如果已超出则调用线程池饱和策略
- 重要参数
  - corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程
  - runnableTaskQueue（任务队列）：用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列
    - ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原则对元素进行排序
    - LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列
    - SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工厂方法    ·Executors.newCachedThreadPool使用了这个队列
    - PriorityBlockingQueue：一个具有优先级的无限阻塞队列
  - maximumPoolSize（线程池最大数量）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如果使用了无界的任务队列这个参数就没什么效果
  - ThreadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字，代码如下。new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
  - RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略
    - AbortPolicy：直接抛出异常
    - CallerRunsPolicy：只用调用者所在线程来运行任务
    - DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务
    - DiscardPolicy：不处理，丢弃掉
    - 当然，也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化存储不能处理的任务
  - keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以，如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率
  - TimeUnit
- 线程池关闭
  - 调用shutdown或shutdownNow方法来关闭线程池
  - 它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止
  - shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表
  - shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true
  - 当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法
- 线程池监控
  - 扩展ThreadPoolExecutor，重写beforeExecute和afterExecute，在这两个方法里分别做一些任务执行前和任务执行后的相关监控逻辑，还有个terminated方法，是在线程池关闭后回调
- 为什么不允许使用Executors创建线程池
  - 这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险

## CAS

- 一个线程将某一内存地址中的数值A改成了B，接着又改成了A，此时CAS认为是没有变化，其实是已经变化过了，而这个问题的解决方案可以使用版本号标识，每操作一次version加1。在java5中，已经提供了AtomicStampedReference来解决问题
- CAS造成CPU利用率增加。之前说过了CAS里面是一个循环判断的过程，如果线程一直没有获取到状态，cpu资源会一直被占用
- 能保证一个共享变量的原子操作。当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。从Java1.5开始JDK提供了**AtomicReference**类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作
- ABA问题的解决办法：在变量前面追加版本号或者时间戳，每次变量更新就把版本号加1，则A-B-A就变成1A-2B-3A

## 死锁

- 死锁产生的条件

  - 互斥:资源的锁是排他性的，加锁期间只能有一个线程拥有该资源。其他线程只能等待锁释放才能尝试获取该资源
  - 请求和保持:当前线程已经拥有至少一个资源，但其同时又发出新的资源请求，而被请求的资源被其他线程拥有。此时进入保持当前资源并等待下个资源的状态
  - 不剥夺：线程已拥有的资源，只能由自己释放，不能被其他线程剥夺
  - 循环等待：是指有多个线程互相的请求对方的资源，但同时拥有对方下一步所需的资源。形成一种循环，类似2)请求和保持。但此处指多个线程的关系。并不是指单个线程一直在循环中等待

- 死锁案例

  - ```java
    public class DeadLockDemo implements Runnable{
    
        public static int flag = 1;
    
        //static 变量是 类对象共享的
        static Object o1 = new Object();
        static Object o2 = new Object();
    
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + "：此时 flag = " + flag);
            if(flag == 1){
                synchronized (o1){
                    try {
                        System.out.println("我是" + Thread.currentThread().getName() + "锁住 o1");
                        Thread.sleep(3000);
                        System.out.println(Thread.currentThread().getName() + "醒来->准备获取 o2");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                    synchronized (o2){
                        System.out.println(Thread.currentThread().getName() + "拿到 o2");//第24行
                    }
                }
            }
            if(flag == 0){
                synchronized (o2){
                    try {
                        System.out.println("我是" + Thread.currentThread().getName() + "锁住 o2");
                        Thread.sleep(3000);
                        System.out.println(Thread.currentThread().getName() + "醒来->准备获取 o2");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                    synchronized (o1){
                        System.out.println(Thread.currentThread().getName() + "拿到 o1");//第38行
                    }
                }
            }
        }
    
        public static  void main(String args[]){
    
            DeadLockDemo t1 = new DeadLockDemo();
            DeadLockDemo t2 = new DeadLockDemo();
            t1.flag = 1;
            new Thread(t1).start();
    
            //让main线程休眠1秒钟,保证t2开启锁住o2.进入死锁
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
    
            t2.flag = 0;
            new Thread(t2).start();
    
        }
    }
    ```

  - 代码中， t1创建，t1先拿到o1的锁，开始休眠3秒。然后 t2线程创建，t2拿到o2的锁，开始休眠3秒。然后 t1先醒来，准备拿o2的锁，发现o2已经加锁，只能等待o2的锁释放。 t2后醒来，准备拿o1的锁，发现o1已经加锁，只能等待o1的锁释放。 t1,t2形成死锁

- 排查死锁

  - jps显示所有当前Java虚拟机进程名及pid
  - jstack打印进程堆栈信息

- 解决办法

  - 死锁一旦发生，我们就无法解决了。所以我们只能避免死锁的发生。 既然死锁需要满足四种条件，那我们就从条件下手，只要打破任意规则即可
    - （互斥）尽量少用互斥锁，能加读锁，不加写锁。当然这条无法避免
    - （请求和保持）采用资源静态分配策略（进程资源静态分配方式是指一个进程在建立时就分配了它需要的全部资源）.我们尽量不让线程同时去请求多个锁，或者在拥有一个锁又请求不到下个锁时，不保持等待，先释放资源等待一段时间在重新请求
    - （不剥夺）允许进程剥夺使用其他进程占有的资源。优先级
    - （循环等待）尽量调整获得锁的顺序，不发生嵌套资源请求。加入超时

# JVM

## 运行时数据区域

- 程序计数器
- 虚拟机栈
  - 线程私有，生命周期和线程相同
  - 方法执行时会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息
  - 每个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程
  - 局部变量表存放了编译器可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用，其中64位长度的long和double类型的数据会占用2个局部变量空间（slot），其余数据只占用1个
  - 局部变量表所需要的内存空间在编译器间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小
  - 如果线程请求的深度大于虚拟机所运行的深度，将会抛出StackOutflowError异常
  - 如果虚拟机可以动态扩展，扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常
- 本地方法栈
  - 本地方法栈是为Java虚拟机的native方法服务的
  - 本地方法栈也会抛出StackOutflowError异常和OutOfMemoryError异常
- 堆
  - 线程共享，与虚拟机生命周期一致
  - 分代收集算法，Java堆细分为新生代和老年代；新生代包括：Eden空间、From Survivor空间、To Survivor空间；
  - Java虚拟机规范规定，Java堆可以处于物理上不连续的内存空间，只要逻辑上是连续的即可
  - 如果堆中没有内存完成实例分配，并且堆也无法在扩展时，将会抛出OutOfMemoryError异常
- 方法区，JDK1.8已经将方法区移至Metaspace，字符串常量移至Java Heap
  - 线程共享
  - 存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据
  - 虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来
  - 当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常
- 运行时常量池
  - 运行时常量区是方法区的一部分。jdk1.8已经移动到Java堆中了
  - 用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放
  - 运行时常量池具备动态性，Java语言并不要求常量一定要编译期才能产生，运行期间也可能将新的常量池放入池中，比如String类中的intern()方法
  - 当常量池无法再申请到内存时会抛出OutOfMemoryError异常
- 直接内存
  - 直接内存不是Java虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域
  - 在jdk1.4新加入的NIO类中，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据
  - 显然，本机内存不会受到Java堆大小的限制，但是还是会受到本机内存大小的限制
  - 在配置虚拟机参数时，会根据实际内存大小设置-Xmx等参数信息，但忽略了直接内存，使得各个内存区域总和大于物理内存限制，从而导致动态扩展时出现OutOfMemoryError异常

## Java8元空间

- 方法区永久代的弊端
  - PermGen space的全称是Permanent Generation space,是指内存的永久保存区域，说说为什么会内存益出：这一部分用于存放Class和Meta的信息，Class在被 Load的时候被放入PermGen space区域，它和存放Instance的Heap区域不同,所以如果你的APP会LOAD很多CLASS的话,就很可能出现PermGen space错误。这种错误常见在web服务器对JSP进行pre compile的时候
  - 如果JVM发现有的类已经不再需要了，它会去回收（卸载）这些类，将它们的空间释放出来给其它类使用。Full GC会进行持久代的回收
- 方法区永久代存储的信息
  - JVM中类的元数据在Java堆中的存储区域
  - Java类对应的HotSpot虚拟机中的内部表示也存储在这里
  - 类的层级信息，字段，名字
  - 方法的编译信息及字节码
  - 变量
  - 常量池和符号解析
- 永久代的大小
  - 它的上限是MaxPermSize，默认是64M
  - 持久代用完后，会抛出OutOfMemoryError "PermGen space"异常。解决方案：应用程序清理引用来触发类卸载；增加MaxPermSize的大小
  - 需要多大的持久代空间取决于类的数量，方法的大小，以及常量池的大小
- 为什么移除永久带？
  - 它的大小是在启动时固定好的——很难进行调优。-XX:MaxPermSize，设置成多少好呢？
  - HotSpot的内部类型也是Java对象：它可能会在Full GC中被移动，同时它对应用不透明，且是非强类型的，难以跟踪调试，还需要存储元数据的元数据信息（meta-metadata）
  - 简化Full GC：每一个回收器有专门的元数据迭代器
  - 可以在GC不进行暂停的情况下并发地释放类数据
  - 使得原来受限于持久代的一些改进未来有可能实现
  - 根据上面的各种原因，永久代最终被移除，方法区移至Metaspace，字符串常量池移至Java Heap
- 移除持久代后，PermGen空间的状况
  - 这部分内存空间将全部移除
  - JVM的参数：PermSize 和 MaxPermSize 会被忽略并给出警告（如果在启用时设置了这两个参数）
- Metaspace的组成
  - Klass Metaspace
    - Klass Metaspace就是用来存klass的，klass是我们熟知的class文件在jvm里的运行时数据结构，不过有点要提的是我们看到的类似A.class其实是存在heap里的，是java.lang.Class的一个对象实例。这块内存是紧接着Heap的，和我们之前的perm一样，这块内存大小可通过-XX:CompressedClassSpaceSize参数来控制，这个参数前面提到了默认是1G，但是这块内存也可以没有，假如没有开启压缩指针就不会有这块内存，这种情况下klass都会存在NoKlass Metaspace里，另外如果我们把-Xmx设置大于32G的话，其实也是没有这块内存的，因为这么大内存会关闭压缩指针开关。还有就是这块内存最多只会存在一块
  - NoKlass Metaspace
    - NoKlass Metaspace专门来存klass相关的其他的内容，比如method，constantPool等，这块内存是由多块内存组合起来的，所以可以认为是不连续的内存块组成的。这块内存是必须的，虽然叫做NoKlass Metaspace，但是也其实可以存klass的内容，上面已经提到了对应场景
  - Klass Metaspace和NoKlass Mestaspace都是所有classloader共享的，所以类加载器们要分配内存，但是每个类加载器都有一个SpaceManager，来管理属于这个类加载的内存小块。如果Klass Metaspace用完了，那就会OOM了，不过一般情况下不会，NoKlass Mestaspace是由一块块内存慢慢组合起来的，在没有达到限制条件的情况下，会不断加长这条链，让它可以持续工作
  - 元空间与永久代之间最大的区别在于：
    - 元空间并不在虚拟机中，而是使用本地内存。因此，元空间的大小仅受本地内存限制
- 元空间参数
  - -XX:MetaspaceSize：初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值
  - -XX:MaxMetaspaceSize：最大空间，默认是没有限制的
  - -XX:MinMetaspaceFreeRatio：在GC之后，最小的Metaspace剩余空间容量的百分比，减少为分配空间所导致的垃圾收集 
  - -XX:MaxMetaspaceFreeRatio：在GC之后，最大的Metaspace剩余空间容量的百分比，减少为释放空间所导致的垃圾收集
  - -verbose参数是为了获取类型加载和卸载的信息
- 元空间特点
  - 充分利用了Java语言规范中的好处：类及相关的元数据的生命周期与类加载器的一致
  - 每个加载器有专门的存储空间
  - 只进行线性分配
  - 不会单独回收某个类
  - 省掉了GC扫描及压缩的时间
  - 元空间里的对象的位置是固定的
  - 如果GC发现某个类加载器不再存活了，会把相关的空间整个回收掉
- 元空间内存分配模型
  - 绝大多数的类元数据的空间都从本地内存中分配
  - 用来描述类元数据的类(klasses)也被删除了
  - 分元数据分配了多个虚拟内存空间
  - 给每个类加载器分配一个内存块的列表。块的大小取决于类加载器的类型; sun/反射/代理对应的类加载器的块会小一些
  - 归还内存块，释放内存块列表
  - 一旦元空间的数据被清空了，虚拟内存的空间会被回收掉
  - 减少碎片的策略
- 元空间内存管理
  - 元空间的内存管理由元空间虚拟机来完成。先前，对于类的元数据我们需要不同的垃圾回收器进行处理，现在只需要执行元空间虚拟机的C++代码即可完成。在元空间中，类和其元数据的生命周期和其对应的类加载器是相同的。话句话说，只要类加载器存活，其加载的类的元数据也是存活的，因而不会被回收掉
  - 准确的来说，每一个类加载器的存储区域都称作一个元空间，所有的元空间合在一起就是我们一直说的元空间。当一个类加载器被垃圾回收器标记为不再存活，其对应的元空间会被回收。在元空间的回收过程中没有重定位和压缩等操作。但是元空间内的元数据会进行扫描来确定Java引用
  - 元空间虚拟机负责元空间的分配，其采用的形式为组块分配。组块的大小因类加载器的类型而异。在元空间虚拟机中存在一个全局的空闲组块列表。当一个类加载器需要组块时，它就会从这个全局的组块列表中获取并维持一个自己的组块列表。当一个类加载器不再存活，那么其持有的组块将会被释放，并返回给全局组块列表。类加载器持有的组块又会被分成多个块，每一个块存储一个单元的元信息。组块中的块是线性分配（指针碰撞分配形式）。组块分配自内存映射区域。这些全局的虚拟内存映射区域以链表形式连接，一旦某个虚拟内存映射区域清空，这部分内存就会返回给操作系统
- 元空间Metaspace 调优
  - 使用-XX:MaxMetaspaceSize参数可以设置元空间的最大值，默认是没有上限的，也就是说你的系统内存上限是多少它就是多少
  - -XX:MetaspaceSize选项指定的是元空间的初始大小，如果没有指定的话，元空间会根据应用程序运行时的需要动态地调整大小
    - MaxMetaspaceSize的调优
      - -XX:MaxMetaspaceSize={unlimited}
      - 元空间的大小受限于你机器的内存
      - 元空间的初始大小是21M——这是GC的初始的高水位线，超过这个大小会进行Full GC来进行类的回收
      - 如果启动后GC过于频繁，请将该值设置得大一些
      - 可以设置成和持久代一样的大小，以便推迟GC的执行时间
    - CompressedClassSpaceSize的调优
      - 只有当-XX:+UseCompressedClassPointers开启了才有效
      - -XX:CompressedClassSpaceSize=1G
      - 由于这个大小在启动的时候就固定了的，因此最好设置得大点
      - 没有使用到的话不要进行设置
      - JVM后续可能会让这个区可以动态的增长。不需要是连续的区域，只要从基地址可达就行；可能会将更多的类元信息放回到元空间中；未来会基于PredictedLoadedClassCount的值来自动的设置该空间的大小
  - 正如前面提到了，Metaspace VM管理Metaspace空间的增长。但有时你会想通过在命令行显示的设置参数-XX:MaxMetaspaceSize来限制Metaspace空间的增长。默认情况下，-XX:MaxMetaspaceSize并没有限制，因此，在技术上，Metaspace的尺寸可以增长到无限空间，而你的本地内存分配将会失败
  - 每次垃圾收集之后，Metaspace VM会自动的调整high watermark，推迟下一次对Metaspace的垃圾收集
  - 这两个参数，-XX：MinMetaspaceFreeRatio和-XX：MaxMetaspaceFreeRatio，类似于GC的FreeRatio参数，可以放在命令行
- 提高GC的性能
  - 如果你理解了元空间的概念，很容易发现GC的性能得到了提升
    - Full GC中，元数据指向元数据的那些指针都不用再扫描了。很多复杂的元数据扫描的代码（尤其是CMS里面的那些）都删除了
    - 元空间只有少量的指针指向Java堆。这包括：类的元数据中指向java/lang/Class实例的指针;数组类的元数据中，指向java/lang/Class集合的指针
    - 没有元数据压缩的开销
    - 减少了根对象的扫描（不再扫描虚拟机里面的已加载类的字典以及其它的内部哈希表）
    - 减少了Full GC的时间
    - G1回收器中，并发标记阶段完成后可以进行类的卸载
- 元空间的问题
  - 元空间虚拟机采用了组块分配的形式，同时区块的大小由类加载器类型决定。类信息并不是固定大小，因此有可能分配的空闲区块和类需要的区块大小不同，这种情况下可能导致碎片存在。元空间虚拟机目前并不支持压缩操作，所以碎片化是目前最大的问题

## Java堆内存分配方法

- 指针碰撞
  - 假设Java堆中内存是绝对规整的，所有用过的内存都放到一边，空闲的放到另一边，中间放着一块指针作为分界点的指示器，那分配内存就仅仅是把那个指针向空闲空间那边移动一段和对象大小相当的距离
- 空闲列表
  - 如果Java堆中的内存不是规整的，已使用的内存和空闲的内存相互交错，那就没有办法简单地使用碰撞指针了，虚拟机就必须维护一个列表，记录哪些内存块是可用的，在分配内存的时候从列表中找到一块足够大的内存空间划分给对象实例，并更新列表上的记录
- 选择哪种分配方式是根据Java堆是否规整来决定的，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定
- 在使用Serial、ParNew等带有Compact过程的收集器时，系统采用的分配算法是指针碰撞
- 使用CMS这种基于Mark-Sweep算法的收集器时，采用的分配算法是采用空闲列表

## 分配内存并发安全

- 对象创建在虚拟机是非常频繁的，即使是修改一个指针所指向的位置，在并发情况下也是不安全的，可能出现正在给对象A分配内存，指针还没来得及修改，对象B同时又使用原来的指针来分配内存
- 解决方法
  - 对分配内存空间的动作进行同步处理，即虚拟机采用CAS配上失败重试的方式保证更新操作的原子性
  - 把内存分配的动作按照线程分在不同的空间中进行，即每个线程在Java堆中预先分配一块小内存，成为本地线程分配缓冲（TLAB）。哪个线程要分配内存，就在哪个线程的TLAB上分配，只有TLAB用完并分配新内存时，才需要同步锁定。虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB 参数来设定

## 对象内存布局

- 对象在内存中存储分为3块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）
  对象头
  - 第一部分：存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。官方称为 Mark Word
  - 第二部分：类型指针，对象执行它的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。查找对象的元数据信息并不一定要经过对象本身？？看后面
  - 如果对象是一个Java数组，那在对象头中还必须用一块记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中却无法确定数组的大小
- 实例数据
  - 存储对象真正有效的数据，也是程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的还是在子类定义的，都需要记录下来
- 对齐填充
  - 不是必须存在的，也没有特别的含义。仅仅是起着占位符的作用。因为Hospot VM的自动内存管理系统要求对象起始地址必须是8个字节的整数倍，就是对象大小必须是8的整数倍
  - 而对象头的大小正好是8的整数倍，因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来不全

## 对象回收策略

- 判断对象是否存活
  - 引用计数算法
    - 给对象添加一个引用计数器，每当有一个地方引用它时，计数器就增加1；当引用失效时，计算取就减少1；任何时刻计数器是0的对象就是不可能再被使用的
    - 会出现对象之间的相互循环引用的问题
  - 可达分析算法
    - GC Roots 链
    - 从GC Roots节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（就是从GC Roots到这个对象不可达）时，则证明这个对象是不可用的
    - 在Java语言中，可作为GC Roots的对象包括
      - 虚拟机栈（栈帧中的本地变量表）中引用的对象
      - 方法区中类静态属性引用的对象
      - 方法区中常量引用的对象
      - 本地方法栈的JNI（Native方法）引用的对象
- 对象引用
  - 强引用（Strong Reference）
    - 类似Object obj = new Object(); 这类引用，只要强引用还存在，垃圾收集器永远不会回收被引用的对象。
  - 软引用（Soft Reference）
    - 当内存不足时，软引用就会被回收掉
    - 可以通过SoftReference类来实现软引用
  - 弱引用（Weak Reference）
    - 无论内存是否充足，当垃圾收集器工作时，弱引用被关联的对象都会回收掉。
      可以通过WeakReference类来实现弱引用
  - 虚引用（Phantom Reference）
    - 是最弱的一种引用关系，一个对象是否有虚引用，完全不会对其生存时间构成影响，也无法通过虚引用取得一个对象实例
    - 为对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。
      可以通过PhantomReference类来实现虚引用。PhantomReference类的get方法 永远返回null
- 对象死亡前期
  - 如果对象在进行可达分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是对象是否有必要执行finalize()方法
  - 当对象没有覆盖finalize()方法或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为 没有必要执行。finalize()只会被执行一次
  - 如果对象被判定有必要执行finalize()方法，对象将会被放置到一个叫做F-Queue队列之中，并在稍后有一个虚拟机自动建立的、低优先级的Finalizer线程异步执行，只是调用而已，并不会等待运行结果；如果等待执行结果的话，有可能这个方法是死循环或者非常耗时，会导致整个内存回收系统瘫痪
- 方法区回收
  - 永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类
  - 废弃常量
    - 当一个常量没有任何引用时，就会被系统清理出常量池
  - 无用的类，类需要同时满足下面3个条件才能算是无用的类
    - 该类所有的实例都已经被回收，也就是Java堆中不能做该类的任何实例
    - 加载该类的ClassLoader已经被回收
    - 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过发射访问该类的方法

## 垃圾收集算法

- 标记-清除
  - 效率低、内存碎片
- 复制
  - 将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden空间和其中一块Survivor。回收时，将Eden和Survivor中还存活的对象一次性第复制到另外一块Survivor空间上，最后清理掉Eden和刚使用过的Survivor空间
  - HotSpot虚拟机默认Eden和Survivor的大小比例是8:1，即每次新生代中可用内存空间为整个新生代容量的90%(80%+10%)，只有10%的内存会被“浪费”
  - 然98%的对象可回收只是一般场景下的数据，没办法保证每次回收都只有不多于10%的对象存活。当Survivor空间不够时，需要依赖其他内存（老年代）进行分配担保
  - 缺点：
    - 在对象存活率较高时就需要进行较多的复制操作，效率将会变低。如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象100%存活的极端情况，所以在老年代一般不能直接使用复制算法
- 标记-整理
  - 老年代使用、让所有存活的对象都向一端移动，然后直接清理掉端边界意外的内存
- 分代收集
  - 在新生代中，每次垃圾收集时发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成功就可以完成收集
  - 在老年代中，因为对象存活率高、没有额外的内存进行分配担保，就必须使用“标记-清除”或者“标记-整理”算法来进行回收

## HotSpot算法的实现

- 枚举根节点
  - GC Roots 链时不可能遍历所有的引用，效率太低，而且GC Root链时必须在一个能确保一致性的快照中进行，即在整个分析期间整个执行系统看起来就像是冻结在某个时间点上，不能出现分析过程中对象引用关系还在不断变化，这点就是导致GC进行时必须停顿所有的Java执行程序（Stop The World）的原因
  - 解决方法：在HotSpot的实现中，使用一组OopMap的数据结构来解决，在类加载完成的时候，HotSpot就把对象内什么偏移量上是什么对象类型的数据计算出来，在JIT编译过程中，也会在特定的位置记录下栈和寄存器中哪些位置是引用的，这样，GC扫描时就可以直接获取数据了
- 安全点（Safepoint）
  - 程序执行过程中并非在所有地方都能停顿下来GC，只有在到达安全点才能暂停，在GC时如何让程序都跑到最近的安全点上停顿下来，解决方案：
    - 抢断式中断
      - 不需要线程的执行代码主动去配合，在GC发生时，首先把所有线程全部中断，如果发现有线程中断的地方不在安全点，就恢复线程，让它跑到安全点，现在几乎没有虚拟机采用抢断式中断
    - 主动式中断
      - 当GC需要中断线程时，不直接对线程操作，仅仅简单设置一个标志，各个线程执行时主动去轮询整个标志，发现中断标志为真时就自己中断挂起
- 安全区域（Safe Region）
  - 如果程序没有分配CPU时间，没有执行的时候，即线程处于sleep或者block状态，这时候线程时无法响应中断请求，就无法到达安全点中断挂起，通过安全区域解决
  - 安全区域是指在一段代码之中，引用关系不会发生变化。在这个区域中的任何地方开始GC都是安全的，安全区域其实是安全点的扩展

## 垃圾收集器

### Serial收集器

- 新生代、复制算法、串行（Stop The World，GC停顿）

### ParNew收集器

- 新生代、复制算法、并行
- Serial的多线程版本，其他和Serial完全一致
- 使用CMS收集老年代时，新生代只能使用ParNew和Serial中的一个
- ParNew收集器是使用-XX:+UseConcMarkSweepGC选项后的默认新生代收集器，也可以使用-XX:+UseParNewGC选项来强制指定它
- 可以使用-XX:ParallelGCThreads参数来限制垃圾收集的线程数

### Parallel Scavenge收集器

- 新生代、复制算法、并行
- 关注吞吐量、吞吐量优先的收集器
- -XX:MaxGCPauseMils 最大垃圾收集停顿时间
  - 单位毫秒
  - GC尽力保证回收时间不超过设定值
  - 吞吐量=运行用户代码的时间/(运行用户代码的时间+垃圾收集的时间)，虚拟机总共运行了100分钟，其中垃圾收集花掉1分钟，吞吐量就是99%
  - 不能简单的以为设置的小一点就能使系统的垃圾收集速度变得更快。GC停顿时间缩短是以牺牲吞吐量和新生代空间来换取的：系统系统把新生代设置的小一些，收集300MB新生代肯定比收集500MB快，这也直接导致了垃圾收集发生的更频繁一些，原来10秒收集一次，每次停顿100毫秒，现在变成了5秒收集一次，每次停顿70毫秒。停顿时间的确在下降，但吞吐量也降下来了
- -XX:GCTimeRatio 吞吐量大小
  - 0-100的取值范围
  - 垃圾收集时间占总时间的比
  - 默认99，即最大允许1%时间做GC
- 注意：上面两个参数是矛盾的，因为停顿时间和吞吐量不可能同时调优，所以只存在一个
- -XX:+UseAdaptiveSizePolicy   GC自适应调节策略
  - 这是一个开关参数，打开后，就不需要手工指定新生的大小（-Xmn）、Eden与Survivor区的比列（-XX:SurvivorRatio）、晋升老年代对象的年龄（-XX:PertenureSizeThreshold）等细节参数，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大吞吐量，这种调节方式称为GC自适应的调节策略
  - 使用GC自适应的调节策略后，只需要设置基本的内存数据（如-Xmx设置最大堆），然后使用-XX:MaxGCPauseMils或者-XX:GCTimeRatio参数给虚拟机一个优化目标就可以了

### Serial Old收集器

- 老年代、串行、标记-整理算法
- Serial的老年代收集器

### Parallel Old收集器

- 老年代、并行、标记-整理算法
- Parallel Scavenge收集器的老年代版本

### CMS收集器

- 以获取最短GC停顿时间为目标的收集器
- 老年代、并行、低停顿、标记-清除（内存碎片）
- 收集过程
  - 初始标记
    - 初始标记仅仅是标记一下GC Roots能直接关联到的对象，速度很快
  - 并发标记
    - 并发标记阶段就是进行GC Roots Tracing的过程
  - 重新标记
    - 重新标记是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录
  - 并发清除
  - 初始标记和并发标记需要GC停顿
  - 由于整个过程中耗时最长的并发标记和并发清除过程收集器都是和用户线程一起工作，所以总体来说，CMS收集器的内存回收过程是与用户线程一起并发执行的

### G1收集器

- 并行与并发、分代收集、复制算法
  - G1能充分利用多个CPU，多核环境下的硬件优势，来缩短停顿时间，G1收集器可以通过并发的方式让Java程序继续执行
- 空间整合：不会产生内存碎片，收集后能提供规则的可用内存。这种特性有利用程序长期运行，分配大对象时也不会应为找不到连续的内存空间而提前触发下一次GC
- 可预测的停顿
- GC不需要额外的内存空间（CMS需要预留额外的内存空间存储浮动垃圾）
- G1 堆结构
  - heap被划分为一个个相等的不连续的内存区域（regions），每个region都有一个分代的角色：eden、survivor、old
  - 每个角色的数量并没有强制的限定，也就是说对每种分代内存的大小可以动态变化
  - G1最大的特点就是高效的执行回收，优先去执行哪些大量对象可回收的区域（region）
  - G1使用了gc停顿可预测的模型，来满足用户设定的gc停顿时间，根据用户设定的目标时间，G1会自动地选择哪些region要清除，一次清除多少个region
  - G1从多个region中复制存活的对象，然后集中放入一个region中，同时整理，清除内存（复制算法）
  - 对比CMS，G1使用复制算法不会产生内存碎片
  - 对比Parallel Scavenge（复制算法）、Parallel Old（标记整理算法），Parallel会对整个区域做整理导致gc停顿会比较长，而G1只是特定地整理几个region
- G1 分区
  - 每个分区都可能是年轻代也可能是老年代，但是同一个时刻只能属于某个代
  - 年轻代、幸存区、老年代这些概念还存在，成为了逻辑上的概念，这样方便复用之前分代框架的逻辑在物理上不需要连续，则带来了额外的好处：有的分区内垃圾对象特别多，有的特别少，G1会有限回收垃圾对象特别多的分区这样可以花费较少的时间来回收这些分区的垃圾，也就是G1名字的由来，即首先收集垃圾最多的分区
- G1 相对于 CMS的优势
  - G1在压缩空间方面有优势
  - G1通过将内存空间分成区域（region）的方式避免内存碎片
  - Eden、Survivor、Old区不再固定，在内存使用效率上更灵活
  - G1可以通过设置预期停顿时间（Pause Time）来控制垃圾收集时间，避免应用雪崩现象
  - G1在回收内存后会马上同时做合并空闲内存的工作，而CMS默认是在（stop the world）的时候做
  - G1会在Young GC中使用，而CMS只能在old区使用
- G1 应用场景
  - 服务端多核CPU、JVM内存占用较大的应用
  - 应用在运行过程中会产生大量内存碎片、需要经常压缩空间
  - 想要更加可控、可预期的GC停顿周期；防止高并发下的应用雪崩现象
- G1收集器跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的的空间大小及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先收回价值最大的Region（也就是Garbage-First名称的由来）。这种使用Region划分内存空间以及优先级的区域回收 方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率
- G1收集器为了避免全扫描整个堆，采取的措施
  - Region之间的对象引用以及其他收集器的新生代和老年代之间的对象引用，虚拟机都是使用Remembered Set来避免全堆扫描的
  - G1中每个Region都有一个与之对应的Remembered Set，虚拟机发现程序在对Reference类型的数据进行读写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用的对象是否处于同一个Region之中（在分代的例子中就是检查是否老年代中的对象引用新生代中的对象），如果是，便通过CardTable把相关引用信息记录到被引用对象所属的Region的Remembered Set之中
  - 当进行垃圾回收时，在GC根节点的枚举范围中加入Remembered Set即可保证不对全堆扫描也不会有遗漏
- G1收集器运行流程
  - 初始标记
    - 初始标记阶段仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS的值，让下一阶段的用户程序并发运行时，能在正确可用的Region中创建对象
    - 需要停顿线程，但是耗时很短
  - 并发标记
    - 并发标记是从GC Roots开始对堆中的对象进行可达性分析，找出存活的对象
    - 耗时较长，但是可以与用户线程并发执行
  - 最终标记
    - 最终标记阶段是为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分标记记录
    - 虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set中，这阶段需要停顿线程，但是可以并行执行
  - 筛选回收
    - 首先对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来指定回收计划

### 区分与搭配

- 年轻代：Serial、ParNew、Parallel Scavenge
- 老年代：Serial Old、Parallel Old、CMS
- G1收集器年轻代老年代都可用
- 搭配使用：
  - Serial - Serial Old、Serial - CMS
  - ParNew - Serial Old、ParNew - CMS
  - Parallel Scavenge - Parallel Old、Parallel Scavenge - Serial Old

## GC日志

## 垃圾收集器参数总结

- -XX:UseSerialGC
  - 虚拟机运行在Client模式下的默认值，打开次开关后，使用Serial + Serial Old的收集器组合进行内存回收
- -XX:UseParNewGC
  - 打开此开关后，使用ParNew + Serial Old的收集器组合进行内存回收
- -XX:UseConcMarkSweepGC
  - 打开此开关后，使用ParNew + CMS + Serial Old收集器组合进行内存回收。Serial Old收集器将作为CMS收集器出现Concurrent Mode Failure失败后的后备收集器使用
- -XX:UseParallelGC
  - 虚拟机运行在Server模式下的默认值；打开开关后，使用Parallel Scavenge + Serial Old的收集器组合进行内存回收
- -XX:UseParallelOldGC
  - 打开此开关后，使用Parallel Scavenge + Parallel Old的收集器组合进行内存回收
- -XX:SurivivorRatio
  - 新生代中的Eden区域与Survivor区域的容量比值，默认值为8，代表Eden:Survivor=8:1
- -XX:PretenureSizeThreshold
  - 直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象将直接在老年代分配
- -XX:MaxTenuringThreshold
  - 晋升到老年代的对象年龄。每个对象在坚持过一次Minor GC之后，年龄就增加1，当超过这个参数值时就进入老年代
- -XX:UseAdaptiveSizePolicy
  - 动态调整Java堆中各个区域的大小以及进入老年代的年龄
- -XX:HandlePromotionFailure
  - 是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden和Survivor区的所有对象存活的极端情况
- -XX:ParallelGCThreads
  - 设置并行GC时进行内存回收的线程数
- -XX:GCTimeRatio
  - GC时间占总时间的比率，默认值是99，即允许1%的GC时间，仅在使用Parallel Scavenge收集器时生效
- -XX:MaxGCPauseMillis
  - 设置GC的最大停顿时间，仅在使用Parallel Scavenge收集器时生效
- -XX:CMSInitiatingOccupancyFraction
  - 设置CMS收集器在老年代空间被使用多少后触发垃圾收集。默认值为68%，仅在使用CMS收集器时生效
- -XX:UseCMSCompactAtFullCollection
  - 设置CMS收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅在使用CMS收集器生效
- -XX:CMSFullGCsBeforeCompaction
  设置CMS收集器在进行若干次垃圾收集后再启动一次内存碎片整理。仅在使用CMS收集器生效。

## 内存分配与回收策略

- 对象优先在Eden区中分配。当Eden区没有足够的空间进行分配时，虚拟机将发起一次Minor GC
- 大对象直接进入老年代
- 长期存活的对象将进入老年代，通过对象年龄判定，可以设置阈值
- 动态对象年龄判定
  - 虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代
  - 如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄
- 空间分配担保
  - 在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间
  - 如果这个条件成立，那么Minor GC可以确保是安全的
  - 如果这个调剂不成立，则虚拟机会查看XX:HandlePromotionFailure设置值是否运行担保失败
  - 如果运行，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小
  - 如果大于，将尝试一次Minor GC，尽管这次Minor GC是有风险的
  - 如果小于，或者XX:HandlePromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC
  - 新生代使用复制收集算法，但为了内存利用率，只是用其中一个Survivor空间来作为轮换备份，因此当出现大量对象在Minor GC后仍然存活的情况（最极端的情况是内存回收后新生代中所有对象都存活），就需要老年代分配担保，把Survivor无法容纳的对象直接进入老年代，前提是老年代还有容纳这些对象的剩余空间，一共有多少对象会活下来在实际完成内存回收之前是无法明确知道的，所以只好取之前每一次回收晋升到老年代容量的平均大小值作为经验值，与老年代的剩余空间进行比较，决定是否进行Full GC来让老年代腾出更多空间
  - 如果某次Minor GC存活后的对象突增，远远高于平均值，依然会导致担保失败，那就只好在失败后重新发起一次Full GC
  - 大部分情况还是会将XX:HandlePromotionFailure打开，避免Full GC过于频繁。

## Jdk监控和故障处理工具

- jps：JVM Process Status Tool，显示指定系统内所有的HotSpot虚拟机进程
- jstat：JVM Statistics Monitoring Tool，用于收集HotSpot虚拟机各方面的运行数据
- jinfo：Configuration Info for Java，显示虚拟机配置信息
- jmap：Memory Map for java，生成虚拟机内存快照
- jhat：JVM Heap Dump Brower，用于分析headdump文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器上查看分析结果
- jstack：Stack Trace for Java，显示虚拟机的线程快照

## 类加载机制

- 类的生命周期

  - 加载
    - 加载时机
      - 创建类的实例
      - 访问某个类或接口的静态变量，或者对该变量赋值
      - 调用类的静态方法
      - 反射 Class.forName("com.")
      - 初始化一个类的子类
      - Java虚拟机启动被标注为启动类的类
  - 连接
    - 验证：确保被加载的类的正确性
    - 准备：为类的静态变量分配内存，并将其初始化为默认值
    - 解析：把类中的符号引用转换为直接引用
  - 初始化：为类的静态变量赋予正确的初始值
    - 当一个类初始化时，会先初始化父类，依次类推
  - 使用
  - 卸载

- 常量在编译阶段会存入到调用这个常量的方法所在类的常量池中，调用类并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。将常量存放到了MyTest的常量池中，之后MyTest和MyParent没有任何关系，即使把MyParent的class文件删除也不影响执行

  - ```java
    public class MyTest {
    			public static void main(String[] args) {
    				System.out.println(MyParent.str);
    			}
    		}
    		class MyParent {
    			public static final String str = "hello world";
    			public static final String str2 = UUID.randomUUID().toString; // 如果引用str2，那么MyParent会初始化，因为编译期无法确定
    			static {
    				System.out.println("MyParent static block");
    			}
    		}
    ```

- 当一个接口初始化时，并不要求其父接口完成初始化

- 类与类加载器

  - 对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的名称空间
  - 比较两个类是否相等，只有在这两个类是有同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就一定不相等
  - 这里的相等是指Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括使用instanceof关键字做对象所属关系判定等情况

- 双亲委派模式

  - 启动类加载器，使用C++语言实现，是虚拟机自身的一部分

  - 其他类加载器，都是继承自抽象类java.lang.ClassLoader

  - 启动类加载器（Bootstrap ClassLoder）

    - 负责加载<JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的类库。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，那直接使用null替代即可

  - 扩展类加载器（Extension ClassLoader）

    - 这个类加载器由sun.misc.Launcher$ExtClassLoader实现，负责加载<JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器

  - 应用类加载器（Application ClassLoader）

    - 这个类加载器由sum.misc.Launcher$Application ClassLoader实现
    - 由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以也称为系统类加载器
    - 它负责加载用户路径（classpath）上所指定的类库，开发者可以直接使用这个类加载器

  - 双亲委派模型的工作过程：

    - 如果一个类加载器收到了类加载器的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一层次的类加载器都是如此，因此所有的请求最终都会传送到顶层的启动类加载器中，只有当父类加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去加载

  - 破坏双亲委派模型

    - 举例：JNDI服务，JNDI现在已经是Java的标准服务了，它的代码有启动类加载器加载，它需要由独立厂商实现并部署在应用程序的ClassPath下的JNDI接口提供者（SPI，Service Provider Interface）的代码，但启动类加载器不可能认识这些代码，怎么办？

  - 线程上下文类加载器（Thread Context ClassLoader）

    - 默认线程上下文类加载器为 系统类加载（App ClassLoader）

    - 在双亲委托模型下，类加载器是由上至下的，即下层的类加载器会委托上层进行加载。但是对于SPI来说，有些接口是Java核心库所提供的，而Java核心库是由启动类加载器来加载的，而这些接口的实现却来自于不同的jar包（厂商提供），Java的启动类加载器是不会加载其他来源的jar包，这样传统的双亲委托模型就无法满足SPI的要求，而通过给当前线程设置上下文类加载器，就可以有设置的上下文类加载器来实现对于接口实现类的加载

    - 这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果线程创建时还未设置，它将会从父线程继承一个，如果应用程序的全局范围内都没有设置过的话，那这个类加载器就是默认类加载器

    - JNDI服务使用这个线程上下文加载器去加载所需要的SPI代码，也就是父类加载器请求子类加载器去完成类加载的动作

    - Java中所有涉及SPI的加载动作基本都采用这种方式，例如：JNDI、JDBC、JCE、JAXB和JBI等

    - 线程上下文类加载器的一般使用模式（获取 - 使用 - 还原）

      - ```java
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        		try {
        			// 自定义的ClassLoader，设置以后 下面的方法就会使用自定义类加载器
        			// 记得还原类加载器，这样不影响后面的操作
        			Thread.currentThread().setContextClassLoader(targetClassLoader); 
        			myMethod();
        		} finally {
        			Thread.currentThread().setContextClassLoader(classLoader);
        		}
        ```

- ServiceLoader

  - MATA-INF/services
  - 通过使用线程上下文类加载器来实现

- 类加载器的命名空间

  - 每个类有自己的命名空间
  - 如果自定义类加载器加载classpath下的类，仍然用的是App ClassLoader类加载器，不会使用自定义的

- 类加载路径

  - System.getProperty("sum.boot.class.path");
  - System.getProperty("java.ext.dirs");
  - System.getProperty("java.class.path");
  - 如果将自定义类放入到根类加载器中，那么这个类就会被根类加载器加载

- Class.forName()和ClassLoader都可以对类进行加载

  - ClassLoader就是遵循双亲委派模型最终调用启动类加载器的类加载器，实现的功能是“通过一个类的全限定名来获取描述此类的二进制字节流”，获取到二进制流后放到JVM中。Class.forName()方法实际上也是调用的CLassLoader来实现的。最后调用的方法是forName0这个方法，在这个forName0方法中的第二个参数被默认设置为了true，这个参数代表是否对加载的类进行初始化，设置为true时会类进行初始化，代表会执行类中的静态代码块，以及对静态变量的赋值等操作。也可以调用Class.forName(String name, boolean initialize,ClassLoader loader)方法来手动选择在加载类的时候是否要对类进行初始化
  - JDBC时通常是使用Class.forName()方法来加载数据库连接驱动。这是因为在JDBC规范中明确要求Driver(数据库驱动)类必须向DriverManager注册自己
  - 我们看到Driver注册到DriverManager中的操作写在了静态代码块中，这就是为什么在写JDBC时使用Class.forName()的原因了

## JVM内存调优参数

- MetaspaceSize
  - 初始化的Metaspace大小，控制元空间发生GC的阈值。GC后，动态增加或降低MetaspaceSize。在默认情况下，这个值大小根据不同的平台在12M到20M浮动。使用Java -XX:+PrintFlagsInitial命令查看本机的初始化参数
- MaxMetaspaceSize
  - 限制Metaspace增长的上限，防止因为某些情况导致Metaspace无限的使用本地内存，影响到其他程序。在本机上该参数的默认值为4294967295B（大约4096MB）
- MinMetaspaceFreeRatio
  - 当进行过Metaspace GC之后，会计算当前Metaspace的空闲空间比，如果空闲比小于这个参数（即实际非空闲占比过大，内存不够用），那么虚拟机将增长Metaspace的大小。默认值为40，也就是40%
  - 设置该参数可以控制Metaspace的增长的速度，太小的值会导致Metaspace增长的缓慢，Metaspace的使用逐渐趋于饱和，可能会影响之后类的加载。而太大的值会导致Metaspace增长的过快，浪费内存
- MaxMetasaceFreeRatio
  - 当进行过Metaspace GC之后， 会计算当前Metaspace的空闲空间比，如果空闲比大于这个参数，那么虚拟机会释放Metaspace的部分空间。默认值为70，也就是70%
- MaxMetaspaceExpansion
  - Metaspace增长时的最大幅度。在本机上该参数的默认值为5452592B（大约为5MB）
- MinMetaspaceExpansion
  - Metaspace增长时的最小幅度。在本机上该参数的默认值为340784B（大约330KB为）
- -Xmx：堆的最大值 -Xms：堆的最小值 -Xmn：设置新生代大小
- -XX:NewRatio：
  - 新生代(eden+2*s)和老年代（不包含永久区）的比值表示 新生代:老年代=1:4，即年轻代占堆的1/5
- -XX:SurvivorRatio
  - 设置两个Survivor区和eden的比表示两个Survivor:eden=2:8，即一个Survivor占年轻代的1/10
- -XX:OnOutOfMemoryError：
  - 可以在内存溢出的时候执行一个脚本，比如可以报警发送一个邮件，重启程序等
  - 例如：-XX:OnOutOfMemoryError=D:/tools/jdk1.7/bin/printstack.bat %p
- -XX:+HeapDumpOnOutOfMemoryError：OOM时导出堆到文件
- -XX:+HeapDumpPath：导出OOM的路径
  - 上面两个例子：-Xmx20m -Xms5m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=d:/a.dump
- -XX:PermSize：永久区的初始大小
- -XX:MaxPermSiz：永久区的最大空间
- -XX:+DisableExplicitGC来禁止RMI调用System.gc
- -XX:MaxDirectMemorySize：直接内存大小

## JIT即时编译

- 热点代码

  - 当虚拟机发现某个方法或代码块的运行特别频繁时，就会把这些代码认定为“热点代码”
  - 被多次调用的方法
    - 一个方法被调用得多了，方法体内代码执行的次数自然就多，成为“热点代码”是理所当然的
  - 被多次执行的循环体
    - 一个方法只被调用过一次或少量的几次，但是方法体内部存在循环次数较多的循环体，这样循环体的代码也被重复执行多次，因此这些代码也应该认为是“热点代码”

- 热点探测

  - 基于采样的热点探测
    - 采用这种方法的虚拟机会周期性地检查各个线程的栈顶如果发现某个（或某些）方法经常出现在栈顶，那这个方法就是“热点方法”
    - 优点：实现简单高效，容易获取方法调用关系（将调用堆栈展开即可）
    - 缺点：不精确，容易因为因为受到线程阻塞或别的外界因素的影响而扰乱热点探测
  - 基于计数器的热点探测：
    - 采用这种方法的虚拟机会为每个方法（甚至是代码块）建立计数器，统计方法的执行次数，如果次数超过一定的阈值就认为它是“热点方法”
    - 优点：统计结果精确严谨
    - 缺点：实现麻烦，需要为每个方法建立并维护计数器，不能直接获取到方法的调用关系
  - HotSpot使用第二种 - 基于计数器的热点探测方法

- 确定了检测热点代码的方式，如何计算具体的次数呢？

  - 方法调用计数器：
    - 这个计数器用于统计方法被调用的次数。默认阈值在 Client 模式下是 1500 次，在 Server 模式下是 10000 次
  - 回边计数器：
    - 统计一个方法中循环体代码执行的次数
  - 达到计数器的阈值会触发即时编译
  - 两个计数器的协作（这里讨论的是方法调用计数器的情况）：
    - 当一个方法被调用时，会先检查该方法是否存在被 JIT（后文讲解） 编译过的版本，如果存在，则优先使用编译后的本地代码来执行。如果不存在已被编译过的版本，则将此方法的调用计数器加 1，然后判断方法调用计数器与回边计数器之和是否超过方法调用计数器的阈值。如果已经超过阈值，那么将会向即时编译器提交一个该方法的代码编译请求
  - 当编译工作完成之后，这个方法的调用入口地址就会被系统自动改成新的，下一次调用该方法时就会使用已编译的版本

- 什么是字节码、机器码、本地代码？

  - 字节码是指平常所了解的 .class 文件，Java 代码通过 javac 命令编译成字节码
  - 机器码和本地代码都是指机器可以直接识别运行的代码，也就是机器指令
  - 字节码是不能直接运行的，需要经过 JVM 解释或编译成机器码才能运行
  - 为什么 Java 不直接编译成机器码，这样不是更快吗？
    - 机器码是与平台相关的，也就是操作系统相关，不同操作系统能识别的机器码不同，如果编译成机器码那岂不是和 C、C++差不多了，不能跨平台，Java 就没有那响亮的口号 “一次编译，到处运行”
    - 之所以不一次性全部编译，是因为有一些代码只运行一次，没必要编译，直接解释运行就可以。而那些“热点”代码，反复解释执行肯定很慢，JVM在运行程序的过程中不断优化，用JIT编译器编译那些热点代码，让他们不用每次都逐句解释执行

- 什么是JIT？

  - 为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成与本地平台相关的机器码，并进行各种层次的优化，完成这个任务的编译器称为即时编译器（Just In Time Compiler），简称 JIT 编译器

- 什么是编译和解释？

  - 编译器：把源程序的每一条语句都编译成机器语言,并保存成二进制文件,这样运行时计算机可以直接以机器语言来运行此程序,速度很快
  - 解释器：只在执行程序时,才一条一条的解释成机器语言给计算机来执行,所以运行速度是不如编译后的程序运行的快的
  - 通过javac命令将 Java 程序的源代码编译成 Java 字节码，即我们常说的 class 文件。这是我们通常意义上理解的编译
  - 字节码并不是机器语言，要想让机器能够执行，还需要把字节码翻译成机器指令。这个过程是Java 虚拟机做的，这个过程也叫编译。是更深层次的编译。（实际上就是解释，引入 JIT 之后也存在编译）
  - Java 需要将字节码逐条翻译成对应的机器指令并且执行，这就是传统的 JVM 的解释器的功能，正是由于解释器逐条翻译并执行这个过程的效率低，引入了 JIT 即时编译技术
  - 不管是解释执行，还是编译执行，最终执行的代码单元都是可直接在真实机器上运行的机器码，或称为本地代码

- 为何 HotSpot 虚拟机要使用解释器与编译器并存的架构？

  - 解释器与编译器两者各有优势
  - 解释器：
    - 当程序需要迅速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间，立即执行
  - 编译器：
    - 在程序运行后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译成本地代码之后，可以获取更高的执行效率

- 在 HotSpot 中，解释器和 JIT 即时编译器是同时存在的，他们是 JVM 的两个组件。对于不同类型的应用程序，用户可以根据自身的特点和需求，灵活选择是基于解释器运行还是基于 JIT 编译器运行。HotSpot 为用户提供了几种运行模式供选择，可通过参数设定，分别为：解释模式、编译模式、混合模式，HotSpot 默认是混合模式，需要注意的是编译模式并不是完全通过 JIT 进行编译，只是优先采用编译方式执行程序，但是解释器仍然要在编译无法进行的情况下介入执行过程。

- 逃逸分析

  - 当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他方法中，称为方法逃逸。甚至可能被外部线程访问到，譬如赋值给类变量或可以在其他线程中访问的实例变量，称为线程逃逸
  - 如果能证明一个对象不会逃逸到方法或线程之外，也就是别的方法或线程无法通过任何途径访问到这个对象，则可以为这个变量进行一些高效的优化：
    - 栈上分配：
      - 将不会逃逸的局部对象分配到栈上，那对象就会随着方法的结束而自动销毁，减少垃圾收集系统的压力
    - 同步消除：
      - 如果该变量不会发生线程逃逸，也就是无法被其他线程访问，那么对这个变量的读写就不存在竞争，可以将同步措施消除掉（同步是需要付出代价的）
    - 标量替换：
      - 如果一个对象不会被外部访问，并且对象可以被拆散的话，真正执行时可能不创建这个对象，而是直接创建它的若干个被这个方法使用到的成员变量来代替。这种方式不仅可以让对象的成员变量在栈上分配和读写，还可以为后后续进一步的优化手段创建条件

- JIT 运行模式

  - 客户端&服务端模式（-client(c1)、-server(c2)）
    - -client时会采用一个叫做C1的轻量级编译器，-server时会采用一个叫做C2的重量级编译器。-server 相对于 -client来说编译更彻底，性能也更好。java -version查看会有一个-server或者-client的标示
    - 以通过在启动java 的命令中，传入参数(-client 或 -server)来选择编译器（C1或C2）。这两种编译器的最大区别就是，编译代码的时间点不一样。client编译器（C1）会更早地对代码进行编译，因此，在程序刚启动的时候，client编译器比server编译器执行得更快。而server编译器会收集更多的信息，然后才对代码进行编译优化，因此，server编译器最终可以产生比client编译器更优秀的代码
  - JVM为什么要将编译器分为client和server，为什么不在程序启动时，使用client编译器，在程序运行一段时间后，自动切换为server编译器？
    - 这种技术是存在的，一般称之为： tiered compilation。Java7 和Java 8可以使用选项-XX:+TieredCompilation来打开（-server选项也要打开）
    - 在Java 8中，-XX:+TieredCompilation默认是打开的

- JVM JIT参数

  - -Xint

    - 全部使用字节码解释执行，这个是最慢的，慢的惊人，通常要比其他方式慢一个数量级左右

  - -Xcomp

    - 纯编译执行，全部被编译成机器码执行，速度是很快的，但是存在一个缺陷，-Xcomp的策略过于简单

  - Xmixed

    - 是一种自适应的方式，有的地方解释执行，有的地方编译执行，具体的策略要依据profile的统计分析来判断

  - -XX:+TieredCompilation

    - 除了纯编译和默认的mixed之外，jvm 从jdk6u25 之后，引入了分层编译。HotSpot 内置两种编译器，分别是client启动时的c1编译器和server启动时的c2编译器，c2在将代码编译成机器代码的时候需要搜集大量的统计信息以便在编译的时候进行优化，因此编译出来的代码执行效率比较高，代价是程序启动时间比较长，而且需要执行比较长的时间，才能达到最高性能；与之相反， c1的目标是使程序尽快进入编译执行的阶段，所以在编译前需要搜集的信息比c2要少，编译速度因此提高很多，但是付出的代价是编译之后的代码执行效率比较低，但尽管如此，c1编译出来的代码在性能上比解释执行的性能已经有很大的提升，所以所谓的分层编译，就是一种折中方式，在系统执行初期，执行频率比较高的代码先被c1编译器编译，以便尽快进入编译执行，然后随着时间的推移，执行频率较高的代码再被c2编译器编译，以达到最高的性能

      作者：worldcbf
      链接：https://www.jianshu.com/p/318617435789
      来源：简书
      著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

- 为什么不全都去编译执行呢？

  - 因为我们程序中有挺大一部分调用频次比较低，并且一次编译的花销要比解释执行大不少，还会有额外的操作。这时去使用JIT就得不偿失了

# 设计模式

- 设计模式原则
  - 单一职责原则
  - 接口隔离原则
    - Spring的接口拆得很细，类可以实现多个接口，可以按照自己的业务来实现不同的接口
  - 依赖倒转原则
    - 高层模块不应该依赖底层模块，二者都应该依赖其抽象
    - 抽象不应该依赖细节，细节应该依赖抽象
    - 核心思想：面向接口编程
    - 使用接口或者抽象类的目的是指定好规范，而不涉及具体任何的操作，把展现细节的任务交给具体的实现类去完成
    - 底层模块尽量都要有抽象类或者接口，或者两者都有，程序更好扩展
    - 变量的声明类型尽量是抽象类或接口，这样我们的变量引用和实际对象间，就存在一个缓冲层，利于程序扩展和优化
    - 实现方式
      - 通过接口方式传递  void play(IPlay play); 这是接口中的一个方法，方法中再传入接口
      - 通过构造方法传递  private IPlay play;  这是类中的一个属性
  - 里式替换原则
    - 继承实际上让两个类的耦合性增强了，在适当的情况下通通过组合来替换继承
  - 开闭原则
    - 对扩展开放，对修改关闭；用抽象构建框架，用实现扩展细节
  - 迪米特法原则
    - 只与朋友通信；成员变量、方法参数、方法返回值中的类称为直接的朋友，而出现在局部变量的类不是直接的朋友，陌生的类最好不要以局部变量表的形式出现在类的内部
    - 被依赖的类不管有多复杂，都尽量封装在类的内部，对外除了提供的public方法，不对外泄露任何信息
  - 合成复用原则
    -  尽量使用合成/聚合的方式，而不是使用继承

# 网络编程

## OSI七层模型

- 应用层
  - HTTP、FTP、SMTP
- 表示层
- 会话层
  - SMTP、DNS
- 传输层
  - TCP、UDP
- 网络层
  - IP、ICMP、ARP
- 数据链路层
- 物理层

## TCP/IP四层模型

- 应用层
- 传输层
- 网络层
- 数据链路层

## TCP粘包和拆包

- 粘包
  - 服务端一共就读到一个数据包，这个数据包包含客户端发出的两条消息的完整信息，这个时候基于之前逻辑实现的服务端就蒙了，因为服务端不知道第一条消息从哪儿结束和第二条消息从哪儿开始，这种情况其实是发生了TCP粘包
- 拆包
  - 服务端一共收到了两个数据包，第一个数据包只包含了第一条消息的一部分，第一条消息的后半部分和第二条消息都在第二个数据包中，或者是第一个数据包包含了第一条消息的完整信息和第二条消息的一部分信息，第二个数据包包含了第二条消息的剩下部分，这种情况其实是发送了TCP拆，因为发生了一条消息被拆分在两个包里面发送了，同样上面的服务器逻辑对于这种情况是不好处理的
- TCP为提高性能，发送端会将需要发送的数据发送到缓冲区，等待缓冲区满了之后，再将缓冲中的数据发送到接收方。同理，接收方也有缓冲区这样的机制，来接收数据
- 发生TCP粘包、拆包主要是由于下面一些原因：
  - 应用程序写入的数据大于套接字缓冲区大小，这将会发生拆包
  - 应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上，这将会发生粘包
  - 进行mss（最大报文长度）大小的TCP分段，当TCP报文长度-TCP头部长度>mss的时候将发生拆包
  - 接收方法不及时读取套接字缓冲区数据，这将发生粘包
- 如何解决拆包粘包？
  - 既然知道了tcp是无界的数据流，且协议本身无法避免粘包，拆包的发生，那我们只能在应用层数据协议上，加以控制。通常在制定传输数据时，可以使用如下方法：
    - 使用带消息头的协议、消息头存储消息开始标识及消息长度信息，服务端获取消息头的时候解析出消息长度，然后向后读取该长度的内容
      - 可以制定，首部固定10个字节长度用来保存整个数据包长度，位数不够补0的数据协议
      - 0000000036{"type":"message","content":"hello"}
      - 可以制定，首部4字节网络字节序unsigned int，标记整个包的长度
      - ****{"type":"message","content":"hello all"}
      - 其中首部四字节*号代表一个网络字节序的unsigned int数据，为不可见字符，紧接着是Json的数据格式的包体数据。
    - 设置定长消息，服务端每次读取既定长度的内容作为一条完整消息
    - 设置消息边界，服务端从网络流中按消息编辑分离出消息内容
      - 假设区分数据边界的标识为换行符"\n"（注意请求数据本身内部不能包含换行符），数据格式为Json，例如下面是一个符合这个规则的请求包
      - {"type":"message","content":"hello"}\n
      - 注意上面的请求数据末尾有一个换行字符(在PHP中用双引号字符串"\n"表示)，代表一个请求的结束。

## TCP

- TCP是全双工的，即客户端在给服务器端发送信息的同时，服务器端也可以给客户端发送信息。而半双工的意思是A可以给B发，B也可以给A发，但是A在给B发的时候，B不能给A发，即不同时，为半双工。 单工为只能A给B发，B不能给A发； 或者是只能B给A发，不能A给B发
- 如果已经建立了连接，但是客户端突然出现故障了怎么办？
  - TCP还设有一个保活计时器，显然，客户端如果出现故障，服务器不能一直等下去，白白浪费资源。服务器每收到一次客户端的请求后都会重新复位这个计时器，时间通常是设置为2小时，若两小时还没有收到客户端的任何数据，服务器就会发送一个探测报文段，以后每隔75秒钟发送一次。若一连发送10个探测报文仍然没反应，服务器就认为客户端出了故障，接着就关闭连接

## TCP三次握手

- 第一次握手
  - SYN=1  seq=J
  - 建立连接时，客户端发送syn包（syn=j）到服务器，并进入SYN_SENT状态，等待服务器确认；SYN：同步序列编号（Synchronize Sequence Numbers）
- 第二次握手
  - SYN=1 ACK=1 ack=J+1 seq=K
  - 服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态
- 第三次握手
  - ACK=1 ack=K+1
  - 客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1），此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手

- 为什么要发送特定的数据包，随便发不行吗？
  - 三次握手的另外一个目的就是确认双方都支持TCP，告知对方用TCP传输
  - 第一次握手：Server 猜测Client可能要建立TCP请求，但不确定，因为也可能是Client乱发了一个数据包给自己
  - 第二次握手：通过ack=J+1，Client知道Server是支持TCP的，且理解了自己要建立TCP连接的意图
  - 第三次握手：通过ack=K+1，Server知道Client是支持TCP的，且确实是要建立TCP连接
- 为什么三次握手，两次不行吗？
  - 假如 A 发出的一个连接请求报文段并没有丢失，而是在某个网络节点滞留了，以致延误到连接释放以后的某个时间节点才到达B。本来这是一个早已经失效的报文段，但 B 收到此失效的连接请求报文段后，就误以为时 A 又发出一次新的连接请求。于是就向 A 发出确认报文段，同意建立连接。假定不采用三次握手，那么只要 B 发出确认，新的连接就建立了
  - 由于现在 A 并没有发出建立连接的请求，因此不会理睬 B 的确认，也不会向 B 发送数据，但 B 却以为新的运输连接已经建立，并一直等待 A 发来数据，B 的许多资源就这样浪费了
  - 采用三次握手的办法可以防止上述问题的发生，例如在刚才的情况下，A 不会向 B 的确认发出确认，B 由于收不到确认，就知道 A 并没有要求建立连接

## TCP四次挥手

- 由于TCP连接时全双工的，因此，每个方向都必须要单独进行关闭，这一原则是当一方完成数据发送任务后，发送一个FIN来终止这一方向的连接，收到一个FIN只是意味着这一方向上没有数据流动了，即不会再收到数据了，但是在这个TCP连接上仍然能够发送数据，直到这一方向也发送了FIN。首先进行关闭的一方将执行主动关闭，而另一方则执行被动关闭
- 第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态
- 第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态
- 第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态
- 第四次挥手：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手
- 为什么建立连接是三次握手，而关闭连接却是四次挥手呢？
  - 因为服务端在LISTEN状态下，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里发送给客户端。而关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送
  - 因为当Server端收到Client端的SYN连接请求后，可以直接发送SYN+ACK报文，其中ACK报文是用来应答的，SYN报文是用来同步的，但是关闭连接时，当Server端收到FIN报文时，很可能并不会立即关闭Socket，所以只能先回复一个ACK报文，高数Client端，“你发送的FIN报文我收到了”，只有等我Server端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送，故需要四次挥手

## HTTPS

- HTTP 是明文传输的，没有加密，不安全，所以就出现了HTTPS
- HTTPS即加密的HTTP，HTTPS并不是一个新协议，而是HTTP+SSL（TLS）。原本HTTP先和TCP（假定传输层是TCP协议）直接通信，而加了SSL后，就变成HTTP先和SSL通信，再由SSL和TCP通信，相当于SSL被嵌在了HTTP和TCP之间
- 协商过程使用非对称加密算法，数据传输使用共享密码加密算法
- 共享密钥加密（对称密钥加密）：
  - 加密和解密同用一个密钥。加密时就必须将密钥传送给对方，那么如何安全的传输呢？
- 公开密钥加密（非对称密钥加密）：
  - 公开密钥加密使用一对非对称的密钥。一把叫做私有密钥，一把叫做公开密钥。私有密钥不能让其他任何人知道，而公开密钥则可以随意发布，任何人都可以获得。使用此加密方式，发送密文的一方使用公开密钥进行加密处理，对方收到被加密的信息后，再使用自己的私有密钥进行解密。利用这种方式，不需要发送用来解密的私有密钥，也不必担心密钥被攻击者窃听盗走
  - 但由于公开密钥比共享密钥要慢，所以我们就需要综合一下他们两者的优缺点，使他们共同使用，而这也是HTTPS采用的加密方式。在交换密钥阶段使用公开密钥加密方式，之后建立通信交换报文阶段则使用共享密钥加密方式
  -  这里就有一个问题，如何证明公开密钥是货真价实的公开密钥。如，正准备和某台服务器建立公开密钥加密方式下的通信时，如何证明收到的公开密钥就是原本预想的那台服务器发行的公开密钥。或许在公开密钥传输过程中，真正的公开密钥已经被攻击者替换掉了。为了解决这个问题，可以使用由数字证书认证机构（CA，Certificate Authority）和其他相关机关颁发的公开密钥证书
  - 接收到证书的客户端可以使用数字证书认证机构的公开密钥，对那张证书上的数字签名进行验证，一旦验证通过，客户端便可以明确两件事：
    - 认证服务器的公开密钥的是真实有效的数字证书认证机构
    - 服务器的公开密钥是值得信赖的
- 工作流程
  - 认证服务器
    - 浏览器内置一个受信任的CA机构列表，并保存了这些CA机构的证书。第一阶段服务器会提供经CA机构认证颁发的服务器证书，如果认证该服务器证书的CA机构，存在于浏览器的受信任CA机构列表中，并且服务器证书中的信息与当前正在访问的网站（域名等）一致，那么浏览器就认为服务端是可信的，并从服务器证书中取得服务器公钥，用于后续流程。否则，浏览器将提示用户，根据用户的选择，决定是否继续。当然，我们可以管理这个受信任CA机构列表，添加我们想要信任的CA机构，或者移除我们不信任的CA机构
  - 认证服务器
    - 浏览器内置一个受信任的CA机构列表，并保存了这些CA机构的证书。第一阶段服务器会提供经CA机构认证颁发的服务器证书，如果认证该服务器证书的CA机构，存在于浏览器的受信任CA机构列表中，并且服务器证书中的信息与当前正在访问的网站（域名等）一致，那么浏览器就认为服务端是可信的，并从服务器证书中取得服务器公钥，用于后续流程。否则，浏览器将提示用户，根据用户的选择，决定是否继续。当然，我们可以管理这个受信任CA机构列表，添加我们想要信任的CA机构，或者移除我们不信任的CA机构
  - 加密通讯
    - 此时客户端服务器双方都有了本次通讯的会话密钥，之后传输的所有Http数据，都通过会话密钥加密。这样网路上的其它用户，将很难窃取和篡改客户端和服务端之间传输的数据，从而保证了数据的私密性和完整性
- 加解密过程 非对称加密+对称加密
  - 用户在浏览器发起HTTPS请求（如 https://www.mogu.com/），默认使用服务端的443端口进行连接
  - HTTPS需要使用一套**CA数字证书**，证书内会附带一个**公钥Pub**，而与之对应的**私钥Private**保留在服务端不公开
  - 服务端收到请求，返回配置好的包含**公钥Pub**的证书给客户端
  - 客户端收到**证书**，校验合法性，主要包括是否在有效期内、证书的域名与请求的域名是否匹配，上一级证书是否有效（递归判断，直到判断到系统内置或浏览器配置好的根证书），如果不通过，则显示HTTPS警告信息，如果通过则继续
  - 客户端生成一个用于对称加密的**随机Key**，并用证书内的**公钥Pub**进行加密，发送给服务端
  - 服务端收到**随机Key**的密文，使用与**公钥Pub**配对的**私钥Private**进行解密，得到客户端真正想发送的**随机Key**
  - 服务端使用客户端发送过来的**随机Key**对要传输的HTTP数据进行对称加密，将密文返回客户端
  - 客户端使用**随机Key**对称解密密文，得到HTTP数据明文
  - 后续HTTPS请求使用之前交换好的**随机Key**进行对称加解
- 加解密过程
  - 用户在浏览器发起HTTPS请求
  - 服务端会返回CA数字证书，证书内包含公钥Pub，私钥存在服务端
  - 客户端严重证书是否是受信CA机构的，同时随机生成客户端RSA公私钥、会话密钥，然后使用服务端的公钥加密发送（客户端私钥、客户端会话密钥）
  - 服务端使用服务器私钥解密数据（得到 客户端公钥、客户端密钥），随机生成会话服务器密钥，然后使用客户端公钥加密发送（服务器会话密钥）
  - 客户端使用私钥解密服务器数据得到 服务器会话密钥
  - 客户端 使用客户端会话秘钥加密后的HTTP数据 发送到服务器
  - 服务器 通过之前的客户端会话密钥解密数据
  - 服务器 使用服务器会话密钥加密后的HTTP数据 发送到客户端
  - 客户端 使用之前的服务器会话密钥解密数据

## 多路复用IO

- select
  - 单个进程可监视的fd数量被限制，即能监听端口的大小有限
  - 对socket进行扫描时时线性扫描，即采用轮询的方法，效率较低
  - 需要维护一个用来存放大量fd数量的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大
- poll
  - 大量的fd的数组被整体复制于用户态和内核地址空间之间，而不管这样的复制有没有意义
  - poll还有一个特点时“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd
- epoll
  - 没有最大并发连接限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）
  - 效率提升，不是轮询的方式，不会随着FD数目的增加效率下降
  - 内存拷贝，利用mmap() 文件映射内存加速与内核空间的消息传递，即poll使用mmap减少复制开销

# Netty

## Netty 的特点是什么？

- 高并发
  - IO 多路复用、Reator模式、BossGroup、WorkGroup
- 传输快
  - Netty 的传输依赖于零拷贝特性，尽量减少不必要的内存拷贝，实现了更高效率的传输
- 封装好
  - Netty 封装了 NIO 操作的很多细节，提供了易于使用调用接口

## Netty 的零拷贝？

- Netty 的接收和发送 ByteBuffer 采用 DIRECT BUFFERS，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行 Socket 读写，JVM 会将堆内存 Buffer 拷贝一份到直接内存中，然后才写入 Socket 中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝
- Netty提供了组合Buffer对象，可以聚合多个ByteBuffer对象，用户可以像操作一个Buffer那样方便的对组合Buffer进行操作，避免了传统通过内存拷贝的方式将几个小Buffer合并成一个大的Buffer
- Netty的文件传输采用了transferTo方法，它可以直接将文件缓冲区的数据发送到目标Channel，避免了传统通过循环write方式导致的内存拷贝问题

## Netty 的线程模型？

- Netty通过Reactor模型基于多路复用器接收并处理用户请求，内部实现了两个线程池，boss线程池和work线程池，其中boss线程池的线程负责处理请求的accept事件，当接收到accept事件的请求时，把对应的socket封装到一个NioSocketChannel中，并交给work线程池，其中work线程池负责请求的read和write事件，由对应的Handler处理
- 单线程模型：所有I/O操作都由一个线程完成，即多路复用、事件分发和处理都是在一个Reactor线程上完成的。既要接收客户端的连接请求,向服务端发起连接，又要发送/读取请求或应答/响应消息。一个NIO 线程同时处理成百上千的链路，性能上无法支撑，速度慢，若线程进入死循环，整个程序不可用，对于高负载、大并发的应用场景不合适
- 多线程模型：有一个NIO 线程（Acceptor） 只负责监听服务端，接收客户端的TCP 连接请求；NIO 线程池负责网络IO 的操作，即消息的读取、解码、编码和发送；1 个NIO 线程可以同时处理N 条链路，但是1 个链路只对应1 个NIO 线程，这是为了防止发生并发操作问题。但在并发百万客户端连接或需要安全认证时，一个Acceptor 线程可能会存在性能不足问题
- 主从多线程模型：Acceptor 线程用于绑定监听端口，接收客户端连接，将SocketChannel 从主线程池的Reactor 线程的多路复用器上移除，重新注册到Sub 线程池的线程上，用于处理I/O 的读写等操作，从而保证mainReactor只负责接入认证、握手等操作


## Netty 高性能表现在哪些方面？

- 心跳
  - 对服务端：会定时清除闲置会话inactive(netty5)，对客户端:用来检测会话是否断开，是否重来，检测网络延迟，其中idleStateHandler类 用来检测会话状态
- 串行无锁化设计
  - 即消息的处理尽可能在同一个线程内完成，期间不进行线程切换，这样就避免了多线程竞争和同步锁。表面上看，串行化设计似乎CPU利用率不高，并发程度不够。但是，通过调整NIO线程池的线程参数，可以同时启动多个串行化的线程并行运行，这种局部无锁化的串行线程设计相比一个队列-多个工作线程模型性能更优
- 可靠性
  - 链路有效性检测：链路空闲检测机制，读/写空闲超时机制；内存保护机制：通过内存池重用ByteBuf;ByteBuf的解码保护；优雅停机：不再接收新消息、退出前的预处理操作、资源的释放操作
- Netty安全性
  - 支持的安全协议：SSL V2和V3，TLS，SSL单向认证、双向认证和第三方CA认证
- 高效并发编程的体现
  - volatile的大量、正确使用；CAS和原子类的广泛使用；线程安全容器的使用；通过读写锁提升并发性能。IO通信性能三原则：传输（AIO）、协议（Http）、线程（主从多线程）
- 流量整型的作用（变压器）
  - 防止由于上下游网元性能不均衡导致下游网元被压垮，业务流中断；防止由于通信模块接受消息过快，后端业务线程处理不及时导致撑死问题
- TCP参数配置
  - SO_RCVBUF和SO_SNDBUF：通常建议值为128K或者256K；SO_TCPNODELAY：NAGLE算法通过将缓冲区内的小封包自动相连，组成较大的封包，阻止大量小封包的发送阻塞网络，从而提高网络应用效率。但是对于时延敏感的应用场景需要关闭该优化算法

## Netty 高性能体现在哪些方面？

- 传输：IO模型在很大程度上决定了框架的性能，相比于bio，netty建议采用异步通信模式，因为nio一个线程可以并发处理N个客户端连接和读写操作，这从根本上解决了传统同步阻塞IO一连接一线程模型，架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。正如代码中所示，使用的是NioEventLoopGroup和NioSocketChannel来提升传输效率
- 协议：采用什么样的通信协议，对系统的性能极其重要，netty默认提供了对Google Protobuf的支持，也可以通过扩展Netty的编解码接口，用户可以实现其它的高性能序列化框架
- 线程：netty使用了Reactor线程模型，但Reactor模型不同，对性能的影响也非常大，下面介绍常用的Reactor线程模型有三种，分别如下：
  - Reactor单线程模型：
    - 单线程模型的线程即作为NIO服务端接收客户端的TCP连接，又作为NIO客户端向服务端发起TCP连接，即读取通信对端的请求或者应答消息，又向通信对端发送消息请求或者应答消息。理论上一个线程可以独立处理所有IO相关的操作，但一个NIO线程同时处理成百上千的链路，性能上无法支撑，即便NIO线程的CPU负荷达到100%，也无法满足海量消息的编码、解码、读取和发送，又因为当NIO线程负载过重之后，处理速度将变慢，这会导致大量客户端连接超时，超时之后往往会进行重发，这更加重了NIO线程的负载，最终会导致大量消息积压和处理超时，NIO线程会成为系统的性能瓶颈
  - Reactor多线程模型：
    - 有专门一个NIO线程用于监听服务端，接收客户端的TCP连接请求；网络IO操作(读写)由一个NIO线程池负责，线程池可以采用标准的JDK线程池实现。但百万客户端并发连接时，一个nio线程用来监听和接受明显不够，因此有了主从多线程模型
  - 主从Reactor多线程模型：
    - 利用主从NIO线程模型，可以解决1个服务端监听线程无法有效处理所有客户端连接的性能不足问题，即把监听服务端，接收客户端的TCP连接请求分给一个线程池。因此，在代码中可以看到，我们在server端选择的就是这种方式，并且也推荐使用该线程模型。在启动类中创建不同的EventLoopGroup实例并通过适当的参数配置，就可以支持上述三种Reactor线程模型

## TCP 粘包/拆包的原因及解决方法？

- 消息定长：FixedLengthFrameDecoder类
- 包尾增加特殊字符分割：行分隔符类：LineBasedFrameDecoder或自定义分隔符类 ：DelimiterBasedFrameDecoder
- 将消息分为消息头和消息体：LengthFieldBasedFrameDecoder类。分为有头部的拆包与粘包、长度字段在前且有头部的拆包与粘包、多扩展头部的拆包与粘包

# Spring

## 容器级别扩展接口

- BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry);
  - 动态添加、注册、移除Bean的BeanDefinition
- BeanFactoryPostProcessor#postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);
  - 允许使用者修改容器中的bean definitions
  - 千万不要进行bean实例化，导致bean被提前实例化
  - 如果在postProcessBeanFactory 方法中执行了bean的初始化，那么会导致@AutoWired和@Resource无法注入bean？
    - 原因：@AutoWired起作用依赖AutowiredAnnotationBeanPostProcessor；@Resource依赖CommonAnnotationBeanPostProcessor；都是BeanPostProcessor，BeanPostProcessors在何处被spring invoke呢？参见registerBeanPostProcessors(beanFactory);在postProcessBeanFactory(beanFactory); 后面被调用，也就是说bean提前初始化的时候，AutowiredAnnotationBeanPostProcessor还没有注册自然就不会被执行到，所以就无法注入bean

## Awre接口

- Awre接口注册时机
  - prepareBeanFactory(beanFactory);  -> beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
  - 在创建容器之后就会注册ApplicationContextAwareProcessor，用来处理Awre接口
- Awre接口调用时机
  - 因为ApplicationContextAwareProcessor是第一个注册，所以会第一个调用
  - EnvironmentAware#setEnvironment(Environment environment);
  - EmbeddedValueResolverAware#setEmbeddedValueResolver(StringValueResolver resolver);
  - ResourceLoaderAware#setResourceLoader(ResourceLoader resourceLoader);
  - ApplicationEventPublisherAware#setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);
  - MessageSourceAware#setMessageSource(MessageSource messageSource);
  - ApplicationContextAware#setApplicationContext(ApplicationContext applicationContext);

## bean 级别扩展接口

- InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation(Class<?> beanClass, String beanName);
  - 目标对象实例化之前调用，可以用来代替原本该生成的目标对象的实例(比如代理对象)
  - 如果该方法的返回值代替原本该生成的目标对象(即 返回不为空)，后续只有postProcessAfterInitialization方法会调用，其它方法不再调用；否则按照正常的流程走
- InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation(Object bean, String beanName);
  - 目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是null
  - 如果该方法返回false，会忽略属性值的设置；如果返回true，会按照正常流程设置属性值
- SmartInstantiationAwareBeanPostProcessor Spring框架内部使用
- SmartInstantiationAwareBeanPostProcessor#predictBeanType(Class<?> beanClass, String beanName);
  - 用于预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null
  - 主要在于BeanDefinition无法确定Bean类型的时候调用该方法来确定类型
- SmartInstantiationAwareBeanPostProcessor#determineCandidateConstructors(Class<?> beanClass, String beanName);
  - 用于选择合适的构造器，比如类有多个构造器，可以实现这个方法选择合适的构造器并用于实例化对象
  - 在postProcessBeforeInstantiation方法和postProcessAfterInstantiation方法之间调用，如果postProcessBeforeInstantiation方法返回了一个新的实例代替了原本该生成的实例，那么该方法会被忽略
- SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference(Object bean, String beanName);
  - 用于解决循环引用问题，提前暴露
- InstantiationAwareBeanPostProcessor#postProcessProperties(PropertyValues pvs, Object bean, String beanName);
  - 对属性值进行修改(这个时候属性值还未被设置，但是我们可以修改原本该设置进去的属性值)
- BeanPostProcessor#postProcessBeforeInitialization(Object bean, String beanName);
  - 这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean
  - 指bean在初始化之前需要调用的方法
- BeanPostProcessor#postProcessAfterInitialization(Object bean, String beanName);
  - 这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean
  - 指bean在初始化之后需要调用的方法
- MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
  - bean在合并Bean定义之后调用
- DestructionAwareBeanPostProcessor#postProcessBeforeDestruction(Object bean, String beanName);
  - 是bean在Spring在容器中被销毁之前调用
- DestructionAwareBeanPostProcessor#requiresDestruction(Object bean);
  - 默认返回true，为true时才会执行postProcessBeforeDestruction方法

## Spring 事件

- ApplicationContextEvent
  - ContextRefreshedEvent
    - ApplicationContext容器初始化或刷新时触发该事件。此处的初始化是指：所有的Bean被成功装载，后处理Bean被检测并激活，所有Singleton Bean 被预实例化，ApplicationContext容器已就绪可用
    - ContextStartedEvent
    - ContextClosedEvent
    - ContextStoppedEvent
    - RequestHandledEvent
      - Web相关事件，只能应用于使用DispatcherServlet的Web应用。在使用Spring作为前端的MVC控制器时，当Spring处理用户请求结束后，系统会自动触发该事件

## Spring Boot 事件

- SpringApplicationEvent
  - ApplicationStartingEvent
  - ApplicationEnvironmentPreparedEvent
  - ApplicationContextInitializedEvent
  - ApplicationPreparedEvent
  - ApplicationStartedEvent
  - ApplicationReadyEvent
  - ApplicationFailedEvent

## Spring Aop

- JDK 动态代理、CGLIB 代理
- 是编译时期进行织入，还是运行期进行织入？
  - 运行期，生成字节码，再加载到虚拟机中，JDK是利用反射原理，CGLIB使用了ASM原理
- 初始化时期织入还是获取对象时织入？
  - 初始化的时候，已经将目标对象进行代理，放入到spring 容器中，用的使用直接拿出来
- spring AOP 默认使用jdk动态代理还是cglib？
  - 要看条件，如果实现了接口的类，是使用jdk。如果没实现接口，就使用cglib
  - 也可以强制指定使用cglib，注解参数：targetSource
- AOP 重要点
  - 增强（Advice）
    - AOP（切面编程）是用来给某一类特殊的连接点，添加一些特殊的功能，那么我们添加的功能也就是增强
  - 切点（PointCut）
    - 一个项目中有很多的类，一个类有很多个连接点，当我们需要在某个方法前插入一段增强（advice）代码时，我们就需要使用切点信息来确定，要在哪些连接点上添加增强
    - 如果把连接点当做数据库中的记录，那么切点就是查找该记录的查询条件
    - 所以，一般我们要实现一个切点时，那么我们需要判断哪些连接点是符合我们的条件的，如：方法名是否匹配、类是否是某个类、以及子类等
  - 连接点（JoinPoint）
    - 连接点就是程序执行的某个特定的位置，如：类开始初始化前、类初始化后、类的某个方法调用前、类的某个方法调用后、方法抛出异常后等
    - Spring 只支持类的方法前、后、抛出异常后的连接点
  - 切面（Aspect）
    - 切面由切点和增强(或引介)组成，或者只由增强（或引介）实现
    - 就是我们定义了标识@Aspect注解的类，这个类就是用来切面处理的
- 反射为什么效率低？
  - 很多方法都是通过Unsafe类调用的，是jni的，JVM无法做出优化
- ASM
  - ASM 能够通过改造既有类，直接生成需要的代码。增强的代码是硬编码在新生成的类文件内部的，没有反射带来性能上的付出

# Spring MVC

## SpringMVC的流程？

- 用户发送请求至前端控制器DispatcherServlet；
- DispatcherServlet收到请求后，调用HandlerMapping处理器映射器，请求获取Handle；
- 处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet；
- DispatcherServlet 调用 HandlerAdapter处理器适配器；
- HandlerAdapter 经过适配调用 具体处理器(Handler，也叫后端控制器)；
- Handler执行完成返回ModelAndView；
- HandlerAdapter将Handler执行结果ModelAndView返回给DispatcherServlet；
- DispatcherServlet将ModelAndView传给ViewResolver视图解析器进行解析；
- ViewResolver解析后返回具体View；
- DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中）
- DispatcherServlet响应用户

# Mybatis

## Mybatis 缓存

- MyBatis 的缓存分为一级缓存和二级缓存,一级缓存放在 session 里面,默认就有,二级缓
  存放在它的命名空间里,默认是不打开的,使用二级缓存属性类需要实现 Serializable 序列化
  接口(可用来保存对象的状态),可在它的映射文件中配置<cache/>

## Mybatis 的插件运行原理

- Mybatis 仅可以编写针对 ParameterHandler、ResultSetHandler、StatementHandler、
  Executor 这 4 种接口的插件，Mybatis 通过动态代理，为需要拦截的接口生成代理对象以实
  现接口方法拦截功能，每当执行这 4 种接口对象的方法时，就会进入拦截方法，具体就是
  InvocationHandler 的 invoke()方法，当然，只会拦截那些你指定需要拦截的方法
- 实现 Mybatis 的 Interceptor 接口并复写 intercept()方法，然后在给插件编写注解，指定
  要拦截哪一个接口的哪些方法即可，记住，别忘了在配置文件中配置你编写的插件

## Mybatis 支持延迟加载？

- Mybatis 仅支持 association 关联对象和 collection 关联集合对象的延迟加载，association
  指的就是一对一，collection 指的就是一对多查询。在 Mybatis 配置文件中，可以配置是否
  启用延迟加载 lazyLoadingEnabled=true|false
- 它的原理是，使用 CGLIB 创建目标对象的代理对象，当调用目标方法时，进入拦截器方
  法，比如调用 a.getB().getName()，拦截器 invoke()方法发现 a.getB()是 null 值，那么就会单
  独发送事先保存好的查询关联 B 对象的 sql，把 B 查询上来，然后调用 a.setB(b)，于是 a 的
  对象 b 属性就有值了，接着完成 a.getB().getName()方法的调用。这就是延迟加载的基本原
  理

## Mybatis Mapper 原理

- Dao接口即Mapper接口。接口的全限名，就是映射文件中的namespace的值；接口的方法名，就是映射文件中Mapper的Statement的id值；接口方法内的参数，就是传递给sql的参数
- Mapper接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为key值，可唯一定位一个MapperStatement。在Mybatis中，每一个标签，都会被解析为一个MapperStatement对象
- Mapper接口里的方法，是不能重载的，因为是使用 全限名+方法名 的保存和寻找策略。Mapper 接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Mapper接口生成代理对象proxy，代理对象会拦截接口方法，转而执行MapperStatement所代表的sql，然后将sql执行结果返回

# Mysql

## 事务属性(ACID)

- 原子性(Atomicy)
- 一致性(Consisteny)
- 隔离性(Isolation)
- 持久性(Durability)

## 隔离级别

- Read Uncommited 
  - 可以读取未提交记录。此隔离级别，不会使用，忽略
- Read Committed (RC)
  - 快照读
  - 针对当前读，RC隔离级别保证对读取到的记录加锁 (记录锁)，存在幻读现象
  - 普通读：总是读取行的最新版本，如果行被锁定了，则读取该行版本的最小一个快照，从数据库理论的角度看，Read Committed事务隔离级别其实违背了是ACID中的I特性，即隔离性
- Repeatable Read (RR)
  - 针对当前读，RR隔离级别保证对读取到的记录加锁 (记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入 (间隙锁)，不存在幻读现象
  - 普通读：总是读取事务开始时的行数据。不会出现Read Committed事务隔离级别下的问题，即不会出现隔离性的问题
- Serializable
  - Serializable隔离级别下，读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用

## 数据库事务问题

- 脏读
  - 事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
- 不可重复读
  - 事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果 不一致
- 幻读
  - 系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读
- 不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表

## MVCC

- MVCC是被Mysql中 事务型存储引擎InnoDB 所支持的;
- 应对高并发事务, MVCC比单纯的加锁更高效;
- MVCC只在 READ COMMITTED 和 REPEATABLE READ 两个隔离级别下工作;其他两个隔离级别够和MVCC不兼容，因为 READ UNCOMMITTED 总是读取最新的数据行, 而不是符合当前事务版本的数据行。而 SERIALIZABLE 则会对所有读取的行都加锁
- MVCC可以使用 乐观(optimistic)锁 和 悲观(pessimistic)锁来实现

## MySQL 乐观锁和悲观锁

- 悲观锁：
  - select .... for update 
  - 并发量大
- 乐观锁：
  - 使用 version 或者 timestamp 进行比较
  - 并发量小
  - CAS
  - update items set inventory=inventory-1 where id=100 and inventory-1>0;

## 聚集索引与非聚集索引

- 聚集索引一个表只能有一个，而非聚集索引一个表可以存在多个

- 聚集索引存储记录是物理上连续存在，而非聚集索引是逻辑上的连续，物理存储并不连续

- 聚集索引表记录的排列顺序和索引的排列顺序一致，所以查询效率快，只要找到第一个索引值记录，其余就连续性的记录在物理也一样连续存放。聚集索引对应的缺点就是修改慢，因为为了保证表中记录的物理和索引顺序一致，在记录插入的时候，会对数据页重新排序

- 非聚集索引制定了表中记录的逻辑顺序，但是记录的物理和索引不一定一致，两种索引都采用B+树结构，非聚集索引的叶子层并不和实际数据页相重叠，而采用叶子层包含一个指向表中的记录在数据页中的指针方式。非聚集索引层次多，不会造成数据重排。这个指针并不是地址，而是主键，通过主键就可以查到数据

- 创建

  - 如果一个主键被定义了，那么这个主键就是作为聚集索引
  - 如果没有主键被定义，那么该表的第一个唯一非空索引被作为聚集索引
  - 如果没有主键也没有合适的唯一索引，那么innodb内部会生成一个隐藏的主键作为聚集索引，这个隐藏的主键是一个6个字节的列，改列的值会随着数据的插入自增
  - Innodb中的每张表都会有一个聚集索引，而聚集索引又是以物理磁盘顺序来存储的，自增主键会把数据自动向后插入，避免了插入过程中的聚集索引排序问题。聚集索引的排序，必然会带来大范围的数据的物理移动，这里面带来的磁盘IO性能损耗是非常大的。 而如果聚集索引上的值可以改动的话，那么也会触发物理磁盘上的移动，于是就可能出现page分裂，表碎片横生

- 如何解决非聚集索引的二次查询问题？

  - 复合索引（覆盖索引）

  - 建立两列以上的索引，即可查询复合索引里的列的数据而不需要进行回表二次查询，如index(col1, col2)，执行下面的语句

  - ```mysql
    	select col1, col2 from t1 where col1 = '213';
    ```

## 为什么MySQL数据库索引选择使用B+树？

- B树在提高了IO性能的同时并没有解决元素遍历的我效率低下的问题，正是为了解决这个问题，B+树应用而生。B+树只需要去遍历叶子节点就可以实现整棵树的遍历。而且在数据库中基于范围的查询是非常频繁的，而B树不支持这样的操作或者说效率太低
- B树在叶子节点和非叶子节点都存放数据
- 所有的数据都在非叶子节点(稠密索引)

# Redis

# Dubbo

# Zookeeper

# 消息队列

# 架构篇

# 搜索引擎

# 性能优化

# 线上问题分析

# 数据结构与算法

# 大数据

# 区块链

# 其他语言

