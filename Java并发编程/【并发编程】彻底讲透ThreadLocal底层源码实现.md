# 一、基本用法

`ThreadLocal` 是一种提供线程局部变量的机制，在`Java`中使用。每个线程都可以独立地改变其副本，而不会影响其他线程的副本。因此，它常用于避免在多线程环境下共享资源的问题

### 1、工作原理

- **初始化**：当你创建一个 `ThreadLocal` 变量时，实际上是在当前线程的 `ThreadLocalMap` 中创建了一个键值对。键是 `ThreadLocal` 实例本身，值是你想要存储的数据
- **存取操作**：当你尝试获取或设置与 `ThreadLocal` 关联的值时，实际上是访问了当前线程内部的 `ThreadLocalMap`。这意味着即使多个线程都持有同一个 `ThreadLocal` 的引用，它们所操作的实际上是不同的对象（因为每个线程都有自己的 `ThreadLocalMap`）
- **线程隔离性**：由于每个线程都有自己独立初始化的变量副本，所以这些副本之间是相互隔离的。这种特性使得 `ThreadLocal` 成为实现线程安全的一种有效手段

### 2、使用场景

1. 用户特定信息的存储：例如，在`Web`应用中存储当前登录用户的会话信息
2. 事务管理：在某些框架中用于追踪事务状态
3. 数据库连接管理：确保每个线程拥有自己的数据库连接实例
4. 格式化器(`Formatter`)和资源池：如日期格式化器等重资源对象的复用

### 3、注意事项

- 内存泄漏问题：`ThreadLocal` 变量如果使用不当可能会导致内存泄漏。这是因为当一个线程长期存活并且它的 `ThreadLocalMap` 没有被清理时，相关的对象也会一直无法被垃圾回收
- 清除数据：建议在不再需要 `ThreadLocal` 保存的数据时，手动调用 `remove()` 方法来删除数据，以帮助GC进行垃圾回收

### 4、示例代码

```java
public class ThreadLocalExample {

    private List<String> messages = Lists.newArrayList();

    private static ThreadLocal<ThreadLocalExample> threadLocalValue = ThreadLocal.withInitial(ThreadLocalExample::new);

    public static void main(String[] args) {
        // 主线程中设置值 Hello
        ThreadLocalExample value = threadLocalValue.get();
        value.messages.add("ABC");

        Runnable task = () -> {
            // 在另外的线程设置值 welcome
            ThreadLocalExample valueTask = threadLocalValue.get();
            valueTask.messages.add("DEF");
            System.out.println("In child thread: " + threadLocalValue.get());
        };

        Thread thread = new Thread(task);
        thread.start();

        try {
            thread.join();    // 等待线程执行完成
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("In main thread: " + threadLocalValue.get());
    }

    @Override
    public String toString() {
        return messages.toString();
    }
}
```

**输出结果：**

```java
In child thread: [DEF]
In main thread: [ABC]
```

此示例展示了如何在主线程和子线程中分别设置和获取 ThreadLocal 变量的不同值。这表明了 ThreadLocal 提供了线程之间的隔离性




.
# 二、源码分析底层实现

## 1、ThreadLocal 构造函数

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}

public ThreadLocal() {
}
```

- 案例中我们用的一个非基础类型的方式来构建我们的 `ThreadLocal` 实例对象，这便于我们理解的会更深刻一些
- `withInitial`方法：创建并返回一个 `SuppliedThreadLocal` 类的实例，同时将传入的 `supplier` 作为构造参数。`SuppliedThreadLocal` 是 `ThreadLocal` 的静态内部类，重写了 `initialValue` 方法，会调用 `supplier` 的 `get` 方法来获取初始值



## 2、ThreadLocal.set 方法

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}


// 获取该线程关联的 ThreadLocalMap 对象
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

`set` **方法的核心逻辑是**：获取当前线程的 `ThreadLocalMap`，若存在就直接存储值，若不存在则创建新的 `ThreadLocalMap` 再存储值。这样每个线程都能拥有自己独立的 `ThreadLocal` 变量副本

**关键步骤：**
1. 调用 `Thread.currentThread()` 方法获取当前正在执行的线程实例
2. 调用 `getMap` 方法，传入当前线程 `t`，获取该线程关联的 `ThreadLocalMap` 对象。`ThreadLocalMap` 是 `ThreadLocal` 的静态内部类，用于存储当前线程的所有 `ThreadLocal` 变量及其对应的值
3. 若 `map` 不为 `null`，表明当前线程已经有了 `ThreadLocalMap`，调用 `map.set(this, value)` 方法，将当前 `ThreadLocal` 对象作为键，传入的 `value` 作为值，存入 `ThreadLocalMap` 中
4. 若 `map` 为 `null`，说明当前线程还没有 `ThreadLocalMap`，调用 `createMap(t, value)` 方法，创建一个新的 `ThreadLocalMap`，并把当前 `ThreadLocal` 对象和 `value` 作为初始键值对存入


.
### （a）createMap 第一次初始化当前线程的 threadLocals 数据

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

主要看下下面的构造逻辑，记住此时是 `ThreadLocalMap` 对象，也就是当前线程下以【`ThreadLocal:数据`】为键值对组合的`map`容器（**`注意：只是可以这么理解，但是跟HashMap还是不同的`**）

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a66f9c7469234535949b77a29f70a985.png)

```java
private static final int INITIAL_CAPACITY = 16;


/**
 * 构造一个新的 ThreadLocalMap，初始包含 (firstKey, firstValue) 键值对
 */
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 初始化一个长度为 INITIAL_CAPACITY 的 Entry 数组，INITIAL_CAPACITY 为 16
    table = new Entry[INITIAL_CAPACITY];
    
    /* 
        通过位运算计算 firstKey 在 table 数组中的索引位置
        firstKey.threadLocalHashCode 是 firstKey 的哈希码
        INITIAL_CAPACITY - 1 保证结果在数组索引范围内
    */
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    
    // 在计算得到的索引位置 i 处创建一个新的 Entry 对象，存储 firstKey 和 firstValue
    table[i] = new Entry(firstKey, firstValue);
    
    // 设置当前 ThreadLocalMap 中存储的键值对数量为 1
    size = 1;
    
    // 根据数组长度设置扩容阈值 (容量的2/3），当键值对数量达到阈值时，需要对数组进行扩容
    setThreshold(INITIAL_CAPACITY);
}
```

此构造方法的主要功能是初始化一个新的 `ThreadLocalMap`，创建一个指定初始容量的数组，将传入的键值对存储到数组的对应位置，设置当前存储的键值对数量，并计算扩容阈值

> **注意**：因为是初始化构建 `ThreadLocalMap` 容器，所以 `table` 数组中是空的，此时计算存储索引 `i` 的时候没有数据冲突，直接通过哈希码取模计算索引存储位置就可以存放了



### （b）set 如果不是第一次设值`【重点逻辑】`

```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    // 通过 key 的 threadLocalHashCode 与 len - 1 进行按位与运算，计算出 key 在哈希表中的初始索引
    int i = key.threadLocalHashCode & (len-1);

    // 遍历的条件是 e != null 的场景，也就是说数组中的索引位已经存在元素的场景
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 那这个种处理的就是 e == null 的场景，那就不存在冲突之说
    tab[i] = new Entry(key, value);
    int sz = ++size;
    
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

`set` 方法的主要逻辑是先尝试找到已存在的 `key` 并更新其值，若遇到过期元素则替换，若未找到则创建新元素，最后检查是否需要清理过期元素和扩容

关键步骤：
- **`i` 值**：通过 `key` 的 `threadLocalHashCode` 与 `len - 1` 进行按位与运算，计算出 key 在哈希表中的初始索引
- **14~28 行 for 循环的逻辑，主要处理 `e != null` 的场景（也就是说是哈希冲突的场景）**：
    - 从初始索引 `i` 开始遍历哈希表，使用线性探测法处理哈希冲突
    - 若当前 `Entry` 的 `key` 与传入的 `key` 相等，说明该 `Entry` 存储的就是当前操作的 `threadLocal`对象，此时直接更新该 `Entry` 的值并返回
    - 若当前 `Entry` 的 `key` 为 `null`，说明该 `Entry` 是过期元素，调用 `replaceStaleEntry` 方法替换过期元素并返回
- 31行：**如果 `e == null` 就会跳出for循环**，此时相当于找到了一个空位，直接放置该新的 `ThreadLocal` 实例数据
- 调用 `cleanSomeSlots` 方法尝试清理一些过期元素，若未清理到过期元素且元素数量达到阈值 `threshold`，则调用 `rehash` 方法对哈希表进行重新哈希和可能的扩容操作

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/8c5932fb6b9b48d7963f166142f3006f.png#pic_center)


### （c）replaceStaleEntry 过期替换

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // 临时存储哈希表中的元素
    Entry e;

    /*  
        备份以检查当前连续段中之前是否存在过期元素
        我们一次性清理整个连续段，以避免由于垃圾回收器批量释放引用而导致的持续增量重新哈希
        记录最早需要清理的过期元素的索引，初始为当前发现的过期元素的索引
    */
    int slotToExpunge = staleSlot;
    // 从当前过期元素位置 - 向前扫描
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len)) {
        // 如果发现更早的过期元素
        if (e.get() == null) {
            // 更新最早需要清理的过期元素的索引
            slotToExpunge = i;
        }
    }

    // 向后查找，要么找到指定的键，要么找到连续段的末尾（null 槽位），取先出现的情况
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        // 获取当前元素的键
        ThreadLocal<?> k = e.get();

        // 如果找到了指定的键
        if (k == key) {
            // 更新该键对应的值
            e.value = value;

            // 将找到的键值对元素与过期元素交换位置，以维持哈希表的顺序
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // 如果最早需要清理的过期元素索引还是初始的过期元素索引
            if (slotToExpunge == staleSlot) {
                // 更新为当前找到键的位置
                slotToExpunge = i;
            }
            // 从最早的过期元素开始清理，并启发式地扫描其他槽位清理过期元素
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果在向前扫描时没有找到过期元素，那么在向后扫描找键的过程中遇到的第一个过期元素
        // 就是当前连续段中仍然存在的第一个过期元素
        if (k == null && slotToExpunge == staleSlot) {
            // 更新最早需要清理的过期元素的索引
            slotToExpunge = i;
        }
    }

    // 如果向后扫描未找到指定的键，则将过期槽位的旧值置为 null
    tab[staleSlot].value = null;
    // 在过期槽位插入新的键值对元素
    tab[staleSlot] = new Entry(key, value);

    // 如果存在其他需要清理的过期元素（最早需要清理的索引不等于当前过期元素的索引）
    if (slotToExpunge != staleSlot) {
        // 从最早的过期元素开始清理，并启发式地扫描其他槽位清理过期元素
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }
}
```

该代码是 `ThreadLocalMap` 类中的 `replaceStaleEntry` 方法，其主要作用是在执行 `set` 操作时，将遇到的过期元素（`stale entry`，即键为 `null` 的元素）替换为指定键的新元素。同时，该方法会清理包含过期元素的连续段（`run`）中的所有过期元素

**关键步骤**：
- 15~23行for循环逻辑：**向前扫描连续段**中最早过期的元素（此处跳出条件：遇到空位结束）
- 26~57行for循环逻辑：**向后扫描连续段**
    - 从 `staleSlot` 的下一个位置开始向后扫描，直到遇到 null 元素
    - 如果找到指定的键 `key`，则更新其值，并将该元素与 `staleSlot` 处的过期元素交换位置
    - 调用 `expungeStaleEntry` 方法清理最早的过期元素，再调用 `cleanSomeSlots` 方法启发式地清理其他过期元素，然后返回
    - 如果遇到新的过期元素且 `slotToExpunge` 仍为 `staleSlot`，则更新 `slotToExpunge`
- `tab[staleSlot] = new Entry(key, value)` ：如果向后扫描未找到相等的`key`，则将 `staleSlot` 处的过期元素置为 `null`，并插入当前要设置的`Entry(key, value)`
- 如果存在其他过期元素（`slotToExpunge` 不等于 `staleSlot`），则调用 `expungeStaleEntry` 方法清理最早的过期元素，再调用 `cleanSomeSlots` 方法启发式地清理其他过期元素


### （d）expungeStaleEntry 清理过期元素`【清理核心】`

```java
/**
 * 通过对位于 staleSlot 和下一个空槽位之间可能发生哈希冲突的元素进行重新哈希，来清除一个过期元素。
 * 此方法还会清除在遇到末尾空槽位之前发现的任何其他过期元素。具体可参考 Knuth 著作的第 6.4 节。
 *
 * @param staleSlot 已知键为 null 的槽位索引
 * @return staleSlot 之后下一个空槽位的索引
 *         （staleSlot 和此槽位之间的所有元素都将被检查并清除）
 */
private int expungeStaleEntry(int staleSlot) {
    // 获取当前的哈希表数组
    Entry[] tab = table;
    // 获取哈希表数组的长度
    int len = tab.length;

    // 清除 staleSlot 位置的过期元素
    // 将该位置元素的值置为 null，帮助垃圾回收
    tab[staleSlot].value = null;
    // 将该位置的元素置为 null
    tab[staleSlot] = null;
    // 哈希表中有效元素的数量减 1
    size--;

    // 重新哈希，直到遇到空槽位
    Entry e;
    int i;
    // 从 staleSlot 的下一个位置开始遍历
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         // 移动到下一个位置
         i = nextIndex(i, len)) {
        // 获取当前元素的键
        ThreadLocal<?> k = e.get();
        if (k == null) {
            // 如果键为 null，说明该元素已过期
            // 将该位置元素的值置为 null，帮助垃圾回收
            e.value = null;
            // 将该位置的元素置为 null
            tab[i] = null;
            // 哈希表中有效元素的数量减 1
            size--;
        } else {
            // 计算该键的正确哈希位置
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                // 如果当前位置不是该键的正确哈希位置
                // 将当前位置的元素置为 null
                tab[i] = null;

                // 与 Knuth 6.4 算法 R 不同，我们必须扫描直到遇到空槽位
                // 因为可能有多个过期元素
                while (tab[h] != null)
                    // 移动到下一个位置
                    h = nextIndex(h, len);
                // 将元素放到正确的哈希位置
                tab[h] = e;
            }
        }
    }
    // 返回下一个空槽位的索引
    return i;
}
```

`expungeStaleEntry` 方法属于 `ThreadLocalMap` 类，其主要目的是清除 `ThreadLocalMap` 中的过期元素（**仅限于元素所在的连续数据区间**），避免内存泄漏，同时通过重新哈希确保元素位于正确的位置，维护哈希表的正常结构

**关键步骤：**
- `tab[staleSlot].value = null` 和 `tab[staleSlot] = null`：将 `staleSlot` 位置的元素值和元素本身置为 `null`，以便垃圾回收
- `size--`：减少 `ThreadLocalMap` 中有效元素的数量
- **for循环内逻辑**：重新哈希并清除过期元素
    - 遍历范围：从 `staleSlot` 的下一个位置开始遍历数组，直到遇到空槽位【连续数据段】
    - `k == null`：执行过期元素清理逻辑
    - `k != null`
        - `int h = k.threadLocalHashCode & (len - 1)`：重哈希，计算该键的正确哈希位置
        - `if (h != i)`：如果当前位置不是该键的正确哈希位置，则从当前i位置先取出来 `tab[i] = null` 放到下一个空位置


> **注意**：该方法在逐步遍历该数据段时，同时也在清理其中的过期元素，这本身会将一个连续的段给拆断成几个。那么此时未过期的元素如果不通过重哈希的方式放置，那么在搜索的时候又按连续的段去搜索就会存在实际存在但是查找不到的问题

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0e271b8d1273456da8ed2e0cfdebc185.png#pic_center)


### （e）cleanSomeSlots 启发式扫描部分元素，查找过期元素

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);    //
    return removed;
}
```

该方法通过对数级别的扫描，在不扫描（可能残留垃圾）和全量扫描（插入操作耗时可能达到 `O(n)`）之间取得平衡，高效地清除部分过期元素，避免内存泄漏

**关键步骤：**
- `i = nextIndex(i, len)`：移动到下一个槽位，`nextIndex` 方法用于处理索引越界，实现环形遍历
- 检查元素是否过期：
    - `if (e != null && e.get() == null)`：如果元素不为空且其键为 `null`，说明该元素已过期
    - `n = len`：若发现过期元素，将 `n` 重新设置为数组长度，以便进行更多的扫描
    - `removed = true`：标记有过期元素被移除
    - `i = expungeStaleEntry(i)`：调用 `expungeStaleEntry` 方法清除当前过期元素，并返回下一个空槽位的索引
- `(n >>>= 1) != 0`：使用无符号右移操作 `>>>` 将 `n` 除以 2，实现对数级别的扫描。当 n 变为 0 时，结束循环
- 返回是否清理过过期元素


### （f）扩容机制 rehash

```java
private void set(ThreadLocal<?> key, Object value) {
    // ...... 省略其他代码 ......
    
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

从上面的代码我们就可以知道，扩容的条件：
- 启发式扫描清理过期元素，未清理到任何数据
- 当前哈希表中的数据量要达到阈值以上，threshold = len * 2 / 3


```java
private void rehash() {
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```
- `expungeStaleEntries` 方法，该方法会遍历整个哈希表，清除所有过期元素。过期元素指的是 `Entry` 中键（`ThreadLocal` 对象）为 `null` 的元素。清除这些元素能释放内存，避免内存泄漏
- `if (size >= threshold - threshold / 4)`
    - `size` 表示当前哈希表中有效的元素数量
    - `threshold` 是哈希表的扩容阈值，当哈希表中元素数量达到该阈值时，通常需要进行扩容操作
    - `threshold - threshold / 4` 相当于 `threshold * 3 / 4`，这里使用更低的阈值来决定是否扩容，目的是避免哈希表在阈值附近频繁扩容和缩容，即避免 “**滞后现象（`hysteresis`）**”
- `resize()`：若当前有效元素数量大于等于 `threshold * 3 / 4`，则调用 `resize` 方法对哈希表进行扩容。`resize` 方法会将哈希表的容量扩大一倍，并重新哈希所有有效元素到新的哈希表中


```java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    
    // 遍历整个哈希表，清除所有过期元素
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);        // 清理过期元素
    }
}
```


```java
private void resize() {
    // 保存旧的哈希表数组
    Entry[] oldTab = table;
    // 获取旧哈希表的长度
    int oldLen = oldTab.length;
    // 计算新哈希表的长度，为旧长度的两倍
    int newLen = oldLen * 2;
    // 创建一个新的 Entry 数组作为新的哈希表
    Entry[] newTab = new Entry[newLen];
    // 记录新表中有效元素的数量
    int count = 0;

    // 遍历旧的哈希表
    for (int j = 0; j < oldLen; ++j) {
        // 获取旧表中当前位置的 Entry
        Entry e = oldTab[j];
        if (e != null) {
            // 获取 Entry 中的键（ThreadLocal 对象）
            ThreadLocal<?> k = e.get();
            if (k == null) {
                // 若键为 null，说明该元素已过期，将值置为 null 帮助 GC
                e.value = null; 
            } else {
                // 计算键在新表中的哈希位置
                int h = k.threadLocalHashCode & (newLen - 1);
                // 处理哈希冲突，若新表中该位置已有元素，则寻找下一个可用位置
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                // 将该 Entry 放入新表的合适位置
                newTab[h] = e;
                // 有效元素数量加 1
                count++;
            }
        }
    }

    // 设置新的扩容阈值
    setThreshold(newLen);
    // 更新哈希表中有效元素的数量
    size = count;
    // 将新的哈希表赋值给 table 变量
    table = newTab;
}
```

`resize` 方法通过创建新的更大容量的哈希表，将旧表中的元素重新哈希并迁移到新表，同时清理过期元素，最后更新哈希表的相关属性，确保 `ThreadLocalMap` 在元素数量增加时仍能保持较好的性能

**关键步骤**：
- **初始化操作**：保存旧的哈希表数组，获取旧表长度，计算新表长度（**扩容一倍**），创建新的哈希表数组，并初始化有效元素计数器
- **遍历旧表**：对旧表中存放的每一个元素检查，未过期的按存放规则插入到新的hash数组中
    - 若元素的键为 `null`：说明该元素已过期，直接清理掉将其值置为 null 以便下次垃圾回收
    - 若元素的键不为 `null`：则计算其在新表中的哈希位置，处理可能的哈希冲突，将元素放入新表的合适位置，并更新有效元素计数器
- **更新属性**：设置新的扩容阈值，更新哈希表中有效元素的数量，将新的哈希表赋值给 table 变量

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/42b0288f6ef94e829d515126aab12f24.png#pic_center)

.
## 3、ThreadLocal.get() 方法

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

- 使用 `Thread.currentThread()` 方法获取当前正在执行的线程实例
- 调用 `getMap` 方法，传入当前线程实例 `t`，获取当前线程关联的 `ThreadLocalMap` 对象。`ThreadLocalMap` 用于存储当前线程中所有 `ThreadLocal` 变量及其对应的值
- 若 `map` 不为 `null`，调用 `map.getEntry(this)` 方法，以当前 `ThreadLocal` 对象作为键，尝试从 `ThreadLocalMap` 中获取对应的 `Entry` 对象
- 若 `Entry` 对象不为 `null`，使用 `@SuppressWarnings("unchecked")` 抑制类型转换警告，将 `Entry` 中的 `value` 强制转换为泛型 `T` 类型，赋值给 `result` 并返回。
- 若 `ThreadLocalMap` 为 `null` 或者 `Entry` 为 `null`，说明当前线程还没有该 `ThreadLocal` 变量的值，调用 `setInitialValue` 方法进行初始化，并返回初始化后的值


### （a）getEntry 查找对应的数据

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 通过键的 threadLocalHashCode 与 table 长度减 1 进行按位与运算，计算出键在 table 中的索引位置
    int i = key.threadLocalHashCode & (table.length - 1);
    // 从 table 中获取该索引位置的 Entry 对象
    Entry e = table[i];
    // 检查获取的 Entry 对象是否不为 null 且其键等于传入的 key
    if (e != null && e.get() == key)
        // 若条件满足，说明直接命中，直接返回该 Entry 对象
        return e;
    else
        // 若未直接命中，调用 getEntryAfterMiss 方法进行后续查找
        return getEntryAfterMiss(key, i, e);
}
```

此处我们知道通过key的哈希值计算的索引，未必找到的就是当前 `ThreadLocal` 实例对应的`Entry`
- 如果说，直接命中，也就是 `e.get() == key` 那就最好不过了
- 否则 `getEntryAfterMiss` 往后查找

### （b）getEntryAfterMiss 哈希索引未直接命中，往后查找

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

其实这个也只是从索引位置开始查找连续数据段



### （c）setInitialValue 如果线程从来没有set过值，需要做初始化

```java
private T setInitialValue() {
    // 调用 initialValue 方法获取线程本地变量的初始值
    T value = initialValue();
    // 获取当前正在执行的线程实例
    Thread t = Thread.currentThread();
    // 调用 getMap 方法，获取当前线程关联的 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    // 若 ThreadLocalMap 对象不为 null，说明当前线程已经有了 ThreadLocalMap
    if (map != null)
        // 将当前 ThreadLocal 对象作为键，初始值作为值，存入 ThreadLocalMap 中
        map.set(this, value);
    else
        // 若 ThreadLocalMap 对象为 null，说明当前线程还没有 ThreadLocalMap，创建一个新的 ThreadLocalMap 并将初始值存入
        createMap(t, value);
    // 返回线程本地变量的初始值
    return value;
}
```

![img.png](img.png)

.

# 三、彩蛋部分

或许看到这里，你还会有疑问。我们从头到尾看下来，到处都在判断存入 `Entry[]` 中的元素 `key` 是否过期，那这个`key`是啥时候被处理为过期的 `null` 值的呢？

## 1、ThreadLocalMap 中的元素，key的过期机制是怎样的？

### （a）弱引用机制让 key 变为过期状态

`ThreadLocalMap` 中的 Entry 类继承自 `WeakReference<ThreadLocal<?>>`，将 `ThreadLocal` 实例作为弱引用的 `key`。当外部对 `ThreadLocal` 实例的强引用都消失后，在垃圾回收时，这些仅存在弱引用的 `ThreadLocal` 实例就会被回收，此时 Entry 里的 `key` 变为 `null`，这个 `Entry` 就成了过期元素

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```




.
.
.

------------
**写文章不容易，如觉得有收获，麻烦老铁们动动发财的小手，随手点个赞，在这里感谢大家的认可**
