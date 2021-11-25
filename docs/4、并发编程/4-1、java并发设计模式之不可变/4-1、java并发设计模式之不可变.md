## 1. 开头  

&emsp;&emsp;并发问题都是写造成的，为了不产生并发问题，我们是不是可以不写呢？  
是的，这里有一个并发设计模式叫做 不变性（Immutability）模，简单来说就是对象一旦被创建之后，状态就不再发生变化，变量一旦被赋值，就不允许修改了（没有写操作）；没有修改操作，也就是保持了不变性  


## 2. java实现不可变类 
&emsp;&emsp;将一个类所有的属性都设置成 final 的，并且只允许存在只读方法，那么这个类基本上就具备不可变性了，把这个类也设置成为final的，这样这个类就可以不被继承，不可变了，这样更加的严格。  
&emsp;&emsp;Java 的 String 就是不可变类。每次修改其实是创建了一个新的String。比如我们可以看value[] 和 replace() 的源码如下：
```
public final class String {
  private final char value[];
  // 字符替换
  String replace(char oldChar, 
      char newChar) {
    // 无需替换，直接返回 this  
    if (oldChar == newChar){
      return this;
    }
 
    int len = value.length;
    int i = -1;
    /* avoid getfield opcode */
    char[] val = value; 
    // 定位到需要替换的字符位置
    while (++i < len) {
      if (val[i] == oldChar) {
        break;
      }
    }
    // 未找到 oldChar，无需替换
    if (i >= len) {
      return this;
    } 
    // 创建一个 buf[]，这是关键
    // 用来保存替换后的字符串
    char buf[] = new char[len];
    for (int j = 0; j < i; j++) {
      buf[j] = val[j];
    }
    while (i < len) {
      char c = val[i];
      buf[i] = (c == oldChar) ? 
        newChar : c;
      i++;
    }
    // 创建一个新的字符串返回
    // 原字符串不会发生任何变化
    return new String(buf, true);
  }
}
```  
从上面的代码可以看出只要做了修改，其实是创建了一个新的对象了。  

## 3. java的享元模式
由上面可知，不可变类在使用过程中，可能产生了大量的对象，如何解决这一问题呢？  
答案是享元模式，也就是用对象池：  
* 创建之前，首先去对象池里看看是不是存在；
* 如果已经存在，就利用对象池里的对象；
* 如果不存在，就会新创建一个对象，并且把这个新创建出来的对象放进对象池里。  
  
比如我们的包装类就是使用的对象池。这里我们看下Long类型内部持有一个缓存池保存了[-128,127]之间的数字。初始化的时候就会放入进去
```
Long valueOf(long l) {
  final int offset = 128;
  // [-128,127] 直接的数字做了缓存
  if (l >= -128 && l <= 127) { 
    return LongCache
      .cache[(int)l + offset];
  }
  return new Long(l);
}
// 缓存，等价于对象池
// 仅缓存 [-128,127] 直接的数字
static class LongCache {
  static final Long cache[] 
    = new Long[-(-128) + 127 + 1];
 
  static {
    for(int i=0; i<cache.length; i++)
      cache[i] = new Long(i-128);
  }
}
```  

**注意：我们不能使用享元模式类型的对象作为锁，因为可能多个线程持有的享元模式对象池可能是同一个，锁可能错乱。**
```
class A {
  Long al=Long.valueOf(1);
  public void setAX(){
    synchronized (al) {
      // 省略代码无数
    }
  }
}
class B {
  Long bl=Long.valueOf(1);
  public void setBY(){
    synchronized (bl) {
      // 省略代码无数
    }
  }
}
```
上面两把锁在对象池中其实是一个对象，会互相影响  


## 4. 注意事项
* 对象的所有属性都是 final 的，并不能保证不可变性；  
  * 因为如果属性是一个普通对象，那么就可以修改
  * 不可变对象虽是不可变的，但是使用的时候如果没有原子性保证，其实也是不安全的，可以使用原子类来保证原子性。
* 不可变对象也需要正确发布。  

如下的代码就是示例用原子类解决了不可变对象引用的原子性问题：  
```
public class SafeWM {

  //不可变class
  class WMRange{
    final int upper;
    final int lower;
    WMRange(int upper,int lower){
    // 省略构造函数实现
    }
  }

  //构建原子类对象传入不可变对象
  final AtomicReference<WMRange> rf = new AtomicReference<>(new WMRange(0,0));

  // 设置库存上限
  void setUpper(int v){
    while(true){
      WMRange or = rf.get();
      // 检查参数合法性
      if(v < or.lower){
        throw new IllegalArgumentException();
      }
      WMRange nr = new WMRange(v, or.lower);
      if(rf.compareAndSet(or, nr)){
        return;
      }
    }
  }
}
```

## 5. 总结  
不可变模式非常普遍，Java 语言里面的 String 和 Long、Integer、Double 等基础类型的包装类都在使用，并且使用了享元模式来缓解创建对象多的问题

