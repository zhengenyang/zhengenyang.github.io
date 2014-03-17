---
layout: post
title: "hashCode的秘密"
date: 2014-03-12 10:18:57 +0800
comments: true
categories: [tech, java]
published: false
---

{% img right /images/blog/hashcode.jpg 300 170 %}

> 实现hashCode方法相对来说比较容易，遵循Josh Bloch的《Effective Java》中的第9条规则就能实现不错的hashCode方法，或者使用 [apache common包](http://commons.apache.org/proper/commons-lang/)（实际上也遵照了《Effective Java》中的规则），然而看似简单的hashCode方法还是有很多值得我们深入了解的地方，有些甚至一直被大多数程序员误解，比如：Object中默认的hashCode实现
> 
> 配图取自：[javarevisited.blogspot.com](http://javarevisited.blogspot.com/2011/10/override-hashcode-in-java-example.html)
> 原图地址：[点击查看](http://4.bp.blogspot.com/-siJKhE4GwWs/Toxk511T9ZI/AAAAAAAAAPQ/iaNYYfjGsCk/s320/1-1232907753jyph.jpg)

### hashCode方法

hashCode方法是用来获取对象的散列值（hash）的方法，被定义在Object类中，因此所有类都具有默认的hashCode实现，但是，当在类中
**覆盖了equals方法时，必须同时覆盖hashCode方法，否则就会违反hashCode的规约，并且不能和基于hash的集合类（如HashMap等）一起正常使用**
默认的hashCode方法被定义为**native**的，不同的VM会有不同的默认实现
``` java Object.hashcode()
public native int hashCode();
```

### 默认的hashCode实现（HotSpot VM）

很多人都对Ojbect类中默认的hashCode实现有着错误的理解，认为默认的hashCode方法是使用对象的内存地址，这很大程度上得归咎于hashCode陈旧的javadoc
``` java
* As much as is reasonably practical, the hashCode method defined by 
* class <tt>Object</tt> does return distinct integers for distinct 
* objects. (This is typically implemented by converting the internal 
* address of the object into an integer, but this implementation 
* technique is not required by the 
* Java<font size="-2"><sup>TM</sup></font> programming language.) 
```
这只是早期VM的hashCode实现方式，随着新的GC方式的引入，对象在内存中的地址并不是固定不变的（比如从新生代被移到老生代），HotSpot VM早已改变了hashCode默认的实现方式，那么默认的hashCode到底是怎么实现的呢？我们来看看HotSpot VM的源码就一目了然了

<!-- more -->
- 由于Object中的hashCode是个native的方法，因此我们来看HotSpot VM的Object.c
``` c src/share/native/java/lang/Object.c
static JNINativeMethod methods[] = {
    {"hashCode",    "()I",                    (void *)&JVM_IHashCode},
    {"wait",        "(J)V",                   (void *)&JVM_MonitorWait},
    {"notify",      "()V",                    (void *)&JVM_MonitorNotify},
    {"notifyAll",   "()V",                    (void *)&JVM_MonitorNotifyAll},
    {"clone",       "()Ljava/lang/Object;",   (void *)&JVM_Clone},
};
```
- 我们看到hashCode对应的注册函数为JVM_IHashCode，这个函数定义在jvm.cpp中
``` c src/share/vm/prims/jvm.cpp
JVM_ENTRY(jint, JVM_IHashCode(JNIEnv* env, jobject handle))
    JVMWrapper("JVM_IHashCode");
    // as implemented in the classic virtual machine; return 0 if object is NULL
    return handle == NULL ? 0 : ObjectSynchronizer::FastHashCode (THREAD, JNIHandles::resolve_non_null(handle)) ;
JVM_END
```
- 调用被交给了ObjectSynchronizer::FastHashCode，这个函数定义在synchronizer.cpp中，由于该函数很长，我们只看关键部分
``` c src/share/vm/runtime/synchronizer.cpp
......

if (mark->is_neutral()) {
    hash = mark->hash();              // this is a normal header
    if (hash) {                       // if it has hash, just return it
      return hash;
    }
    hash = get_next_hash(Self, obj);  // allocate a new hash code
    temp = mark->copy_set_hash(hash); // merge the hash code into header
    // use (machine word version) atomic operation to install the hash
    test = (markOop) Atomic::cmpxchg_ptr(temp, obj->mark_addr(), mark);
    if (test == mark) {
      return hash;
    }
    // If atomic operation failed, we must inflate the header
    // into heavy weight monitor. We could add more code here
    // for fast path, but it does not worth the complexity.
}

......

```
- 我们看到，JVM先从对象的头部（header）获取该对象的hash值，如果能取到则直接返回，否则调用get_next_hash函数生成hash值，并经过处理后保存到对象头部，接着看get_next_hash函数（也在synchronizer.cpp中）
``` c src/share/vm/runtime/synchronizer.cpp
......

static inline intptr_t get_next_hash(Thread * Self, oop obj) {
  intptr_t value = 0 ;
  if (hashCode == 0) {
     // This form uses an unguarded global Park-Miller RNG,
     // so it's possible for two threads to race and generate the same RNG.
     // On MP system we'll have lots of RW access to a global, so the
     // mechanism induces lots of coherency traffic.
     value = os::random() ;
  } else
  if (hashCode == 1) {
     // This variation has the property of being stable (idempotent)
     // between STW operations.  This can be useful in some of the 1-0
     // synchronization schemes.
     intptr_t addrBits = intptr_t(obj) >> 3 ;
     value = addrBits ^ (addrBits >> 5) ^ GVars.stwRandom ;
  } else
  if (hashCode == 2) {
     value = 1 ;            // for sensitivity testing
  } else
  if (hashCode == 3) {
     value = ++GVars.hcSequence ;
  } else
  if (hashCode == 4) {
     value = intptr_t(obj) ;
  } else {
     // Marsaglia's xor-shift scheme with thread-specific state
     // This is probably the best overall implementation -- we'll
     // likely make this the default in future releases.
     unsigned t = Self->_hashStateX ;
     t ^= (t << 11) ;
     Self->_hashStateX = Self->_hashStateY ;
     Self->_hashStateY = Self->_hashStateZ ;
     Self->_hashStateZ = Self->_hashStateW ;
     unsigned v = Self->_hashStateW ;
     v = (v ^ (v >> 19)) ^ (t ^ (t >> 8)) ;
     Self->_hashStateW = v ;
     value = v ;
  }

  value &= markOopDesc::hash_mask;
  if (value == 0) value = 0xBAD ;
  assert (value != markOopDesc::no_hash, "invariant") ;
  TEVENT (hashCode: GENERATE) ;
  return value;
}

......

```
- 看到了吗？秘密就在这，HotSpot VM提供了多达6种hashCode的生成算法
    - 0：生成伪随机数（默认实现）
    - 1：对象的内存地址和伪随机数异或
    - 2：固定返回1（仅用作测试）
    - 3：全局递增序列
    - 4：使用对象的内存地址
    - 5：使用线程相关状态运用Marsaglia's xor-shift算法生成
- 可见，默认的hashCode并不是什么内存地址，而是用伪随机数生成算法生成的伪随机数（每次生成以后保存在对象头里），不过在Java8中，默认实现将改用上述第6种算法
- HotSpot VM提供了-XX:hashcode参数来让用户选择使用何种算法生成默认的hashCode，比如：java -XX:hashcode=2 YourClass 将使得Object.hashcode方法始终返回1
- 默认的hashCode是延迟创建的，也就是说不是对象创建时默认的hashCode值就被生成出来并写入对象头中了，而是在第一次使用到时才生成，这里所谓的第一次使用包括
    1. 如果没有覆盖Object的hashCode实现，那么在调用对象的hashCode方法时会调用上述算法生成默认的hashCode并写入对象头
    2. 另一种情况是，调用了System类的identityHashCode(Object x)方法

### System.identityHashCode

这是一个很少被使用的方法，它的作用是返回对象的默认hashCode，无论hashCode方法是否被覆盖，如果参数为null，则返回0

``` java
public class HashCodeTest {
    public static void main(String[] args) {
    
    String str = new String("hello world");
    System.out.println("String's hashCode: " + str.hashCode());
    System.out.println("System.identityHashCode: " + System.identityHashCode(str));
}
```
将上述程序运行两遍，得到以下输出
``` java
// 第一次输出：
// String's hashCode: 1794106052
// System.identityHashCode: 772722277

// 第二次输出： 
// String's hashCode: 1794106052
// System.identityHashCode: 171508166
```
可见，hashCode方法返回的是String类中覆盖后计算出来的值，System.identityHashCode方法返回的是Object类中的默认实现（伪随机数）

### hash碰撞（hash collision）

当两个不相等的对象（或数据）被映射到同一个hash值时，称之为发生了hash碰撞，如下图中的"John Smith" 和 "Sandra Dee"就发生了hash碰撞，它们都被映射为02

{% img /images/blog/hash.png 450 300 %}
> 图片取自：http://en.wikipedia.org/wiki/Hash_function
> 原图地址：[点击查看](http://upload.wikimedia.org/wikipedia/commons/thumb/5/58/Hash_table_4_1_1_0_0_1_0_LL.svg/500px-Hash_table_4_1_1_0_0_1_0_LL.svg.png)  

那么int型的hashcode在理论上的碰撞概率是否非常小呢？毕竟2^32是4,294,967,296，事实果真如此吗？根据生日悖论（[Birthday problem](http://en.wikipedia.org/wiki/Birthday_paradox)） —— 随机选择23个人，其中有2个人生日相同（同月同日）的概率超过50%，虽然int型的hashcode有2^32个可选值，但只要hash的对象达到77,163个，那么发生hash冲突的概率就将超过50%，有兴趣的读者可以参看[这篇文章](http://preshing.com/20110504/hash-collision-probabilities/)

### hash碰撞对HashMap的影响

hash碰撞有很多应用，比如：[彩虹表（Rainbow Table）](http://en.wikipedia.org/wiki/Rainbow_Table)，在这里我们来看看Java中，hash碰撞对HashMap会造成怎样的影响，先来看看HashMap的示意图

{% img /images/blog/hashmap.png 450 300 %}
> 图片取自：http://en.wikipedia.org/wiki/Hash_table
> 原图地址：[点击查看](http://upload.wikimedia.org/wikipedia/commons/thumb/d/d0/Hash_table_5_0_1_1_1_1_1_LL.svg/500px-Hash_table_5_0_1_1_1_1_1_LL.svg.png)

可以看到，"John Smith" 和 "Sandra Dee"发生hash碰撞时，HashMap将这两个元素都放入152号Hash桶里（bucket），同一个Hash桶里的元素再用数组存储，极端情况，当HashMap里所有的元素都发生碰撞时，HashMap就退化成数组了，时间复杂度将从O(1)变成O(n)  

绝大部分的Web服务器（如：Tomcat），都会将请求数据自动保存到HashMap中，以便后续能通过getParameter(String name)等方法获取请求参数，由于用户可以构造请求参数，这样恶意用户就可以通过构造大量的会导致hash冲突的请求参数来发起拒绝服务攻击

这个问题在2003年已被发现，包括Java、Ruby、PHP、Python等众多语言均被波及，Java语言至今未修复该问题  
具体可参见：[这里](http://www.ocert.org/advisories/ocert-2011-003.html)和[这里](https://www.nruns.com/_downloads/advisory28122011.pdf)  
> A Tomcat 6.0.32 server parses a 2 MB string of colliding keys in about 44 minutes of i7 CPU time, so an attacker with about 6 kbit/s can keep one i7 core constantly busy  

在Tomcat中，可以通过设置maxParameterCount（默认为10000）或者maxPostSize（默认为2M）参数来限制请求参数的个数和大小，详见：[Tomcat文档](http://tomcat.apache.org/tomcat-6.0-doc/config/http.html)  
在nginx中，可以通过设置client_max_body_size（默认为1M）参数来限制客户端请求的大小，详见：[nginx文档](http://wiki.nginx.org/HttpCoreModule#client_max_body_size)

### 等效子串（Equivalent substrings）

也许你会觉得要构造大量的会导致hash冲突的请求参数并不是件容易的事，然而事情并非如此  
一些hash函数具备一种被称为等效子串（Equivalent substrings）的特性，即：如果hash(str1)==hash(str2)，那么所有在相同位置包含str1和str2的字符串的hash值也相等，如：hash(astr1b)=hash(astr2b)、hash(33str144)==hash(33str244)......  
String类的hashcode函数正好具备这个特性，因此，只要我们找到一对会发生hash碰撞的字符串，如：Ea和FB、St和TU，那么我们就能轻易的构造出1Ea2和1FB2、11Ea2和11FB2、aSt和aTU等等会导致hash冲突的字符串

### 参考资料
[1. ](id:1)[《Effective Java》by Josh Bloch](http://www.amazon.com/Effective-Java-Edition-Joshua-Bloch/dp/0321356683)   
[2. ](id:2)[Hash Collision Probabilities](http://preshing.com/20110504/hash-collision-probabilities/)   
[3. ](id:3)[http://www.ocert.org/advisories/ocert-2011-003.html](http://www.ocert.org/advisories/ocert-2011-003.html)  
[4. ](id:4)[https://www.nruns.com/_downloads/advisory28122011.pdf](https://www.nruns.com/_downloads/advisory28122011.pdf)