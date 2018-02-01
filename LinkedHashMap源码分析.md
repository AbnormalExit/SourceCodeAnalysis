##LinkedHashMap源码分析
@(源码阅读)

###LinkedHashMap概述
LinkedHashMap是HashMap的一个子类，在保留HashMap插入顺序的基础上还增加了顺序的变更，当操作LinkedHashMap的时候书序会发生改变。LinkedHashMap重写了HashMap的get(Object key)方法，在这个方法中增加调用了 afterNodeAccess(e)方法，该方法会对双链表进行一次排序。

LinkedHashMap和HashMap区别：
LinkedHashMap 内部维护的是一个双链表，HashMap内部维护的是一个哈希表。

###LinkedHashMap的实现
####类结构
```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

####成员变量
```java
    /**
     * The head (eldest) of the doubly linked list.
     */
    transient LinkedHashMapEntry<K,V> head;

    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient LinkedHashMapEntry<K,V> tail;

    /**
     * 
     *这个链接哈希映射的迭代排序方法：访问顺序为true，插入顺序为false。
     * @serial
     */
    final boolean accessOrder;
```
Entry是继承了HashMap的Node

```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```
####初始化
和HashMap一样也是有四个初始化方法，在初始化的时候默认访问顺序为 插入时候的顺序，重写了HashMap的reinitialize()方法初始化head和tail
```java
   // overrides of HashMap hook methods

    void reinitialize() {
        super.reinitialize();
        head = tail = null;
    }
```
这样就完成双链表头和尾的初始化

####存储数据
LinkedHashMap并未重写HashMap的put方法，而是重写了HashMap的afterNodeAccess方法。注：在HashMap的源码中可以看出HashMap为LinkedHashMap暴露了三个方法。

```java
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```
这个方法就是当我们顺序是访问顺序 accessOrder =true的时候，将最近访问的元素放到链表末尾。

####获取数据
LinkedHashMap获取数据重写了HashMap的get方法

```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```
由此可以看出，获取的数据还是和HashMap一样根据getNode获取，这里只是增加了afterNodeAccess的调用，这样就能够对数据进行一次排序，最近访问的元素放到链表的末尾。

####删除数据
LinkedHashMap的删除数据和HashMap也是一样的，不同的就是重写了afterNodeRemoval方法对数据进行一次排序

```java

    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }

```
####简单实践
这里写一个测试案例来验证以上分析的。

```java
public class Main {

    public static void main(String[] args) {
        HashMap<String, String> hashMap = new HashMap<String, String>();
        hashMap.put("1", "A");
        hashMap.put("2", "B");
        hashMap.put("3", "C");
        hashMap.put("4", "D");
        hashMap.put("5", "E");
        hashMap.put("6", "F");
        hashMap.put("7", "G");
        System.out.println("***********HashMap**********");
        printmap(hashMap);
        System.out.println("访问之后");
        hashMap.get("3");
        hashMap.get("5");
        printmap(hashMap);
        System.out.println("***********HashMap**********");
        //仿照LruCache初始化LinkedHashMap
        LinkedHashMap<String, String> map = new LinkedHashMap<String, String>(0, 0.75f, true);
        map.put("1", "A");
        map.put("2", "B");
        map.put("3", "C");
        map.put("4", "D");
        map.put("5", "E");
        map.put("6", "F");
        map.put("7", "G");
        System.out.println("***********LinkedHashMap**********");
        //初始化的顺序
        printmap(map);
        //访问其中数据3,5
        map.get("3");
        map.get("5");
        System.out.println("访问map中数据之后的顺序");
        printmap(map);
        System.out.println("***********LinkedHashMap**********");
    }

    /**
     * 遍历LinkedHashMap
     *
     * @param map
     */
    private static void printmap(Map<String, String> map) {
        Iterator<Map.Entry<String, String>> iterator = map.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, String> entry = iterator.next();
            System.out.println(entry.getKey() + "->" + entry.getValue());
        }
    }
}
```
输出结果为

```
***********HashMap**********
1->A
2->B
3->C
4->D
5->E
6->F
7->G
访问之后
1->A
2->B
3->C
4->D
5->E
6->F
7->G
***********HashMap**********
***********LinkedHashMap**********
1->A
2->B
3->C
4->D
5->E
6->F
7->G
访问map中数据之后的顺序
1->A
2->B
4->D
6->F
7->G
3->C
5->E
***********LinkedHashMap**********
```

####总结
以上简单分析了LinkeHashMap的实现原理，所以我们可以根据自己的需求选择用LinkedHashMap和HashMap,如果对顺序只要插入顺序的就用HashMap,如果对顺序有要求的就用LinkedHashMap。