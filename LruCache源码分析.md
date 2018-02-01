###LruCache<K,V>源码分析
@(源码阅读)

###概述
LRU 是Least Recently Used的缩写。内部原理其实就是使用一个LinkedHashMap对数据进行内存缓存，LinkedHashMap可以进行数据访问排序，当每次去访问双链表的时候都会将最近访问的数据移动到链表的尾部，当内存达到缓存的最大值的时候就可以删除链表最顶端不常用的数据。

###构造函数
构造函数需要传递一个缓存的最大值。
```java
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```
以上代码可以看出，LruCache需要的是基于访问顺序的排序，所以初始化的时候accessOrder=true。

###成员变量

```java
    private int size; //当前cache的大小
    private int maxSize;//cache最大大小

    private int putCount; //put的次数
    private int createCount;//create的次数
    private int evictionCount;//回收的次数
    private int hitCount;//命中的次数
    private int missCount;//未命中次数
```

###缓存数据

```java
    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }
```
添加数据的时候进行了一下几部操作
1. 获取添加数据的大小
```java
   protected int sizeOf(K key, V value) {
        return 1;
    }
```
源码的这里永远都是返回的1，所以当我们自己实现缓存的时候需要重写这个方法，返回我们所缓存数据的大小。
添加了数据之后又去获取了一次数据，检测如果数据为空更改size大小。
2. 移除长时间未使用的缓存数据，直到缓存数量小于或等于最大缓存的值
```java
    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```

###获取数据

```java
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         */

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }
```

由以上代码可以看出，当根据key获取到值的时候直接返回，当获取不到值的时候回去调用create(key)方法创建一个值，但是我们用LruCache通常用来缓存图片，所以当获取不到值的时候就需要到文件或者是网络获取，这里就可以不用重写create方法。

###使用案例
使用LruCache我们可以编写自己的内存缓存类，可以用来缓存图片。
```java
public class MemoryCache {
    private int maxSize = 4 * 1024 * 1024;//4mb
    LruCache<String, Bitmap> bitmapCache;

    private static MemoryCache mInstance = null;

    private MemoryCache() {
        bitmapCache = new LruCache<String, Bitmap>(maxSize) {
            @Override
            protected int sizeOf(String key, Bitmap value) {
                return value.getByteCount();
            }
        };
    }

    public static MemoryCache getInstance() {
        if (mInstance == null) {
            synchronized (MemoryCache.class) {
                if (mInstance == null) {
                    mInstance = new MemoryCache();
                }
            }
        }
        return mInstance;
    }

    public void putCacheBitmap(String key, Bitmap bitmap) {
        if (getCacheBitmap(key) == null) {
            bitmapCache.put(key, bitmap);
        }
    }

    public Bitmap getCacheBitmap(String key) {
        Bitmap bitmap = bitmapCache.get(key);
        return bitmap;
    }
}
```

###总结
通过以上的 代码分析以及简单的代码实现可以了解LruCache的原理，归纳起来也就下面几点。
（1）构造函数LruCache时提供一个总的缓存大小；
（2）重写sizeOf方法，对存入map的数据大小进行自定义测量；
（3）根据需要，决定是否要重写entryRemoved()和create(key)方法；
（4）使用LruCache提供的put和get方法进行数据的缓存
