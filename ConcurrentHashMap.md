

## 变量

### sizeCtl

```java
// 当sizeCtl==-1时, 表示table正在初始化中
// 当sizeCtl==-(1+n)时, 表示有n个线程在对table进行resize扩容
// 当sizeCtl>0, table==null时, 则表示table的初始化大小
// 当sizeCtl>0, talbe !=null 时, 则表示table下一次进行扩容时的阈值
private transient volatile int sizeCtl;
```

## 方法

### initTable

```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // sizeCtl小于0, table在初始化中或者扩容中
            // 释放cpu时间片
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // cas, 将sizeCtl设置为-1
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                    // 必须再次判断table是否初始化
                    // 假设T1,T2线程执行到9行
                    // T1 cas成功, 将sizeCtl设置为-1
                    // T2 cas失败, 然后进入3行的while循环, 执行到6行
                    // T1 执行17行-25行, 将sizeCtl设置为sc, 通过22行计算后sc为12, table此时已经初始化了
                    // T2 执行6行, 此时sizeCtl为12, 则执行9行, 将sizeCtl设置为-1, 执行到17行, 校验table是否初始化, 此时T1 已经完成了table的初始化, 所以只需要执行25行, 将sizeCtl这是为12.// 所以当table完成初始化之后, sizeCtl始终为 n-(n>>>2), 即为0.75*capacity
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);  // 0.75 * n
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

### put

```java
public V put(K key, V value) {
        return putVal(key, value, false);
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    // 无限循环, 直到在循环中将新节点成功放入table中退出
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        // 初始化table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // i为新节点的桶坐标, 如果此时table中i处为null
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 通过CAS将新节点放到i坐标处, 如果放置成功, 则退出for循环
            // 否则, 此时i处已经被别的线程放置了一个新节点, 此时继续往下执行
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                   // no lock when adding to empty bin
        }
        // i处头结点hash为-1
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else if (onlyIfAbsent // check first node without acquiring lock
                 && fh == hash
                 && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                 && (fv = f.val) != null)
            return fv;
        else {
            V oldVal = null;
            // 对头节点进行加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // 判断hash值是否大于等于0
                    if (fh >= 0) {
                        binCount = 1; // 节点个数
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                // 判断是否要转化为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```