---
layout: post
title: "equals-没你想得那么简单"
date: 2014-03-10 14:41:12 +0800
comments: true
categories: [tech, java]
published: false
---

{% img right /images/blog/equals.jpg 300 170 %}

> 对于每个Java程序员来说，说起**equals**方法都是再熟悉不过了，大多数人觉得实现equals方法是小菜一碟，然而实际情况并非如此，在Java社区中，对于如何正确地实现equals方法有着相当多的争议，在实际应用中，不正确的equlas实现往往会产生诡异的、难以排查的问题
>      
> 配图取自：[nicodewet](http://nicodewet.com/)
> 原图地址：[点击查看](http://nicodewet.files.wordpress.com/2012/10/nicodewet-com_equals.jpg)

### 什么是equals方法

为了文章的完整性， 先赘述一下什么是equals方法：通俗地讲，equals方法就是用来判断两个对象是否相等的，equals方法被定义在**Object**类中，因此，所有的类都具有默认的equals实现

### 何时需要覆盖equals方法

之前说到equals方法是用来比较两个对象是否相等的，因此，是否需要自己实现equals方法，取决于如何定义**相等**，以及超类（Super Class）的实现是否已经满足了子类对相等的定义  
**Object**类中提供的默认equals实现是用双等号（==）来判断两个对象是否相等，这意味着默认的equals实现对**相等**的定义是：两个对象在内存中拥有相同的地址。毫无疑问，这是一个合理的默认实现：两个对象在内存中的地址相同说明是同一个对象

- 对象类型
	- 值类型（Value Type）  
	对象所包含的内容至关重要，两个对象是否相等取决于所包含的内容，这类对象被称为**值类型**对象，例如：*String*、*BigDecimal*
	- 实体类型（Entity Type）  
	与值类型相对的是实体类型，对象是否相等与所包含的内容无关，例如：*Thread*、*FileInputStream*
- 值类型需要自己实现equals方法
	- 因为值类型对象是否相等取决于对象包含的内容，内存地址相等并不代表对象相等，因此Object类默认实现的equals方法不适用，需要自己实现，相反的，实体类型则不需要
- 对于值类型，如果有超类（非Object类），并且超类实现的equals方法适用于子类，则子类无需再实现equals方法
	- 例如：*ArrayList*和*LinkedList*都继承了超类*AbstractList*的equals方法  

<!-- more -->
### equals方法规约

了解了什么情况需要自己实现equals方法后，接下来我们先看看实现equals方法需要遵守的规约

- **非空性（non-nullity）**  
对任意非空对象x，x.equals(null)为false
	
- **自反性（reflexive）**  
对任意非空对象x，x.equals(x)为true
	
- **对称性（symmetric）**  
对任意非空对象x和y，当且仅当x.equals(y)为true时，y.equals(x)也为true
	
- **一致性（consistent）**  
对任意非空对象x和y，如果equals方法中用到的字段内容没有被修改，那么无论何时调用x.equals(y)方法，都应该获得一致的结果
	
- **传递性（transitive）**  
对任意非空对象x、y和z，如果x.equals(y)为true并且y.equals(z)为true，则x.equals(z)也为true，反之亦然

### 遵守规约并不容易 
稍微有点数学基础的人都不难理解以上规则，甚至可能觉得遵守以上规则再自然不过，很容易实现，可事实真的如此吗？我们来看一些反例

- JDK中java.net.URL类中的equals实现
``` java java.net.URL
public boolean equals(Object obj) {
	if (!(obj instanceof URL))
		return false;
	URL u2 = (URL)obj;
	return handler.equals(this, u2);
}
```
handler.equals最终会调用到java.net.URLStreamHandler
	
``` java java.net.URLStreamHandler
protected boolean hostsEqual(URL u1, URL u2) {
	InetAddress a1 = getHostAddress(u1);
	InetAddress a2 = getHostAddress(u2);
	// if we have internet address for both, compare them
	if (a1 != null && a2 != null) {
		return a1.equals(a2);
		// else, if both have host names, compare them
	} else if (u1.getHost() != null && u2.getHost() != null) 
		return u1.getHost().equalsIgnoreCase(u2.getHost());
	else
		return u1.getHost() == null && u2.getHost() == null;
}
```
可以看到，JDK中对两个URL的比较最终会转换为对IP地址的比较，由于URL对应的IP地址可能会发生变更，很显然，这样的equals方法是不符合**一致性**规约的，另外，获取IP的过程是阻塞的，调用这样的equals方法也会严重影响程序的性能

- JDK中java.sql.Timestamp类中的equals实现
``` java java.sql.Timestamp
public boolean equals(java.lang.Object ts) {
	if (ts instanceof Timestamp) {
		return this.equals((Timestamp)ts);
	} else {
		return false;
	}
}

public boolean equals(Timestamp ts) {
	if (super.equals(ts)) {
		if  (nanos == ts.nanos) {
			return true;
		} else {
			return false;
		}
	} else {
		return false;
	}
}
```
由于Timestamp类集成自java.util.Date类，我们再看看Date类的equals实现
``` java java.util.Date
public boolean equals(Object obj) {
	return obj instanceof Date && getTime() == ((Date) obj).getTime();
}
```
看出问题了吗？没有？没关系，看看下面的测试代码
``` java TestTimestampEquals
public static void main(String[] args) {
	long now = System.currentTimeMillis();
	Date date = new Date(now);
	Timestamp timestamp = new Timestamp(now);
	System.out.println("date.equals(timestamp): " + date.equals(timestamp));
	System.out.println("timestamp.equals(date): " + timestamp.equals(date));
}
// output￼
// date.equals(timestamp): true
// timestamp.equals(date): false
```
看到了吧，违反了**对称性**规约，事实上Timestamp的javadoc中也特别注明了这个问题
``` java
/** 
* Note: This method is not symmetric with respect to the 
* <code>equals(Object)</code> method in the base class.
*/
```

- 《Program Development in Java￼》中的equals示例（Page 182：Figure 7.16）
``` java Point3
public class Point3 extends Point2 {
	private int z; // the z coordinate

	/* 该方法由自己添加 */
	public Point3(int x, int y, int z) {
		super(x, y);
		this.z = z;
	}
    
	public boolean equals(Object p) { // overriding definition 
		if (p instanceof Point3) return equals((Point3) p); 
		return super.equals(p); 
	}

	public boolean equals(Point2 p) { // overriding definition 
		if (p instanceof Point3) return equals((Point3) p); 
		return super.equals(p); 
	}

	public boolean equals(Point3 p) { // extra definition 
		if (p == null || z != p.z) return false; 
		return super.equals(p);  
	}

}
```
*Point2代表二维的点，包含x和y两个属性，当且仅当x和y都相等时，equals方法返回true*
``` java Point2示意代码
public class Point2 {
    private final int x;
    private final int y;

    public Point2(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public boolean equals(Object p) {
    	if (this == p) {
    		return true;
    	}
    	if (!(p instanceof Point2)) {
    		return false;
    	}
    	Point2 point2 = (Point2) p;
    	return x == point2.x && y == point2.y;
    }
}
```
来看下面的测试代码
``` java TestPoint3Equals
public static void main(String[] args) {
	Point3 point3_1 = new Point3(0, 0, 1);
	Point2 point2 = new Point2(0, 0);
	Point3 point3_2 = new Point3(0, 0, -1);
	System.out.println("point3_1.equals(point2): " + point3_1.equals(point2));
	System.out.println("point2.equals(point3_2): " + point2.equals(point3_2));
	System.out.println("point3_1.equals(point3_2): " + point3_1.equals(point3_2));
}
// output
// point3_1.equals(point2): true
// point2.equals(point3_2): true
// point3_1.equals(point3_2): false
```
看到没，违反了**传递性**规约

###问题出在哪
一般来说，**非空性**、**自反性**和**一致性**是比较容易满足的（只要不像URL类那样在equals方法中引入不确定的因素），容易出问题的是**对称性**和**传递性**，而导致问题的原因是什么呢？没错，是**继承**！
    
以Point2和Point3为例，从Point2的角度来说，它只能看到x和y两个属性，两个对象只要x和y都相等，就认为是相等的，因此，对Point2来说，(0, 0, 1)和（0， 0）以及(0, 0)和(0, 0, -1)是相等的，而对于Point3，它需要额外比较z是否相等，因此，对Point3来说，(0, 0, 1)和(0, 0, -1)显然不等 
   
Joshua Bloch在《Effective Java》中说道：**There is no way to extend an instantiable class and add a value component while preserving the equals contract**（对一个可实例化的类来说，没有办法在扩展该类时既增加值组件（指参与equals比较，如Point3中的z）又遵守equals规约）
  
由此可见，为了遵守equals规约，我们有3个选择

1. 让(超)类不可继承（定义为final）
2. 让超类不可实例化（定义为抽象类）
3. 禁止子类在equals方法中引入新的属性，即禁止子类覆盖(override)equals方法（在超类中将equals方法定义为final）
    
至此，我们知道了造成equals方法违反规约的主要原因是因为继承引入了混合类型比较（mixed-type comparisons），第1种和第2种选择，正是为了消除混合类型（第1种消除子类、第2种消除超类对象），而第3种选择虽然允许混合类型比较，但实际上比较时将子类都“退化为“超类，这导致(0, 0, 1)和(0, 0, -1)的比较结果为true

### 一定要比较子类怎么办
- **使用组合（composition）代替继承**  
按照Joshua Bloch在《Effective Java》中的建议，**Favor composition over inheritance**（组合优于继承），在《Effective Java》中，Joshua Bloch给出了这种权宜之计的示例，这里我们仍然以Point3为例
``` java Point3New
public class Point3New {

	private final Point2 point2;
	private final  int z;

	public Point3New(int x, int y, int z) {
		point2 = new Point2(x, y);
		this.z = z;
	}

	public Point2 asPoint2() {
		return point2;
	}

	@Override
	public boolean equals(Object obj) {
		if (!(obj instanceof Point3New)) {
			return false;
		}
		Point3New point3New = (Point3New) obj;
		return this.point2.equals(point3New.point2) && this.z == point3New.z;
	}
}
```

- **比较默认值**  
在实际应用中，并不是所有的继承都能用组合替代，有时候我们不得不使用继承或者从设计的角度讲继承会更合理，这种让我们无法使用上述方法，但我们还可以用另一种变通的方法：比较默认值。比如：假设Point3中的z属性默认值为0，那么我们认为(0, 0)和(0, 0, 0)相等，(0, 0)和(0, 0, 1)不等
``` java Point2D
public class Point2D {
	
	private int x = 0;
	private int y = 0;

	public Point2D(int x, int y) {
		this.x = x;
		this.y = y;
	}

	@Override
	public boolean equals(Object obj) {
		if (this == obj) {
			return true;
		}
		if (!(obj instanceof Point2D)) {
			return false;
		}
		if (obj.getClass() == Point2D.class) {  // 2个对象都是超类对象，只需要比较超类的属性即可
			Point2D point2D = (Point2D) obj;
			return this.x == point2D.x && this.y == point2D.y;
		} else {    // obj为子类对象，交换比较对象（调用子类的equals方法，多态）
			return obj.equals(this);
		}
	}
}

```
``` java Point3D
public class Point3D extends Point2D {

	private int z = 0;

	public Point3D(int x, int y, int z) {
		super(x, y);
		this.z = z;
	}

	@Override
	public boolean equals(Object obj) {
		if (obj.getClass() == Point3D.class) {  // 2个对象都是子类对象，比较子类的属性(z)
			Point3D point3D = (Point3D) obj;
			if (this.z != point3D.z) {
				return false;
			}
		} else {    // obj是超类对象，判断this的z属性是否是默认值0
			if (this.z != 0) {
				return false;
			}
		}
		return super.equals(obj);
	}
}
```
**为了便于理解，以上程序假设只有2级继承，且做了简化，仅用于演示，不可直接使用**，更完整的示例请参阅参考资料 [3](#3)  

### 正确实现equals方法的步骤
- 在equals方法上使用@Override注解
``` java
@Override
public boolean equals(Object obj) {
```
- 使用==检查是否同一个对象引用
``` java
if (this == obj) {
	return true;
}
```
- 使用**instanceof**进行类型判断
``` java
/* 不需要判空，因为null instanceof any class始终为false
if (obj == null) {
	return false;
}
*/
if (!(obj instanceof Point2D)) {
	return false;
}
```
- 把对象转换为正确的类型
``` java
Point3D point3D = (Point3D) obj;
```
- 比较每一个重要的属性
``` java
if (this.z != point3D.z) {
	return false;
}
```

### 一些要注意的点
- 把类定义为final的（避免继承），或至少把equals方法定义为final的（避免子类覆盖equals方法并引入新的属性）
- 坚持用instanceof做类型判断，而不要用getClass()，因为这会违反[Liskov原则](http://en.wikipedia.org/wiki/Liskov_substitution_principle)
``` java
// 不要这么做！
if (obj == null || obj.getClass() != getClass()) {
	return false;
}
```
- float和double属性使用Float.compare(float, float)和Double.compare(double, double)进行比较，而不是直接用==（因为Float.NaN == Float.NaN为false，而0.0f == -0.0f为true，具体参见Float和Double的javadoc）
- 无论什么时候，覆盖了equals方法都必须覆盖hashcode方法
- 不要声明参数类型不是Object的equals方法
``` java
// 不要这么做！
public boolean equals(Point2 point2) {
	...
}
```

### 参考资料
[1. ](id:1)[《Effective Java》by Josh Bloch](http://www.amazon.com/Effective-Java-Edition-Joshua-Bloch/dp/0321356683)  
[2. ](id:2)[Secrets of equals() - Part 1](http://www.angelikalanger.com/Articles/JavaSolutions/SecretsOfEquals/Equals.html)  
[3. ](id:3)[Secrets of equals() - Part 2](http://www.angelikalanger.com/Articles/JavaSolutions/SecretsOfEquals/Equals-2.html)