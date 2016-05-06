---
layout:     post
title:      "Android DiskLruCache源码分析"
subtitle:   "深入解析DiskLruCache"
date:       2016-05-05
author:     "Woody"
catalog: true
tags:
    - Android
---

## 简介

缓存对于提升一个App的性能至关重要，对于需要使用大量图片的App来说，Android官方推荐使用LruCache+DiskLruCache对图片进行两级缓存，这样当用户重新打开App时，不需要再从网络上获取图片。DiskLruCache是一套磁盘缓存方案，采用LRU算法。

## 基本概念

DiskLruCache以key-value的形式在磁盘上缓存，支持一对多，线程安全。不支持多进程同时访问一个缓存目录。一般App在SD卡的缓存目录在`/sdcard/Android/data/<your packagename>/cache/`。DiskLruCache在缓存目录下会生成一个日志文件`journal`,记录了所有对缓存的操作，数据文件名称以`key.index`方式存储在缓存目录，index指key-value对应关系中一对多的情况value的下标,如果key-value是一对一的情况，那么一般数据文件名称以`key.0`的方式存储。如下图所示

![](http://7xtfm0.com2.z0.glb.clouddn.com/journal1.png)

## journal文件解读

journal文件包含journal文件头+日志，如下图所示

![](http://7xtfm0.com2.z0.glb.clouddn.com/journal2.png)

文件前五行组成一个完整的journal头

1. 第一行是一个字符串常量`libcore.io.DiskLruCache`，标识版权信息
2. 第二行代表DiskLruCache缓存的版本号`1`
3. 第三行代表App的版本号也就是versionCode为`1`
4. 第三行代表key-value是1对N的关系，此例中`N=1`
5. 第五行是空行，与日志内容分隔

日志内容一行代表一条操作记录，格式为`status key value[0].length ... value[N-1].length`,用空格分隔,status代表文件状态，如下

- DIRTY 脏数据，代表正在对此条数据进行操作,**每条DIRTY日志后面都应该是一条CLEAN或者REMOVE日志，否则将删除此key对应的文件**
- CLEAN 代表已经成功写入缓存的数据，可供读取
- READ 代表读取一条数据
- REMOVE 代表删除一条数据

**journal文件是整个DiskLruCache的核心，在使用过程中要对journal文件进行大量的操作**

## 创建缓存

DiskLruCache的构造方法是private的，创建一个DiskLruCache实例需要调用静态方法`open`

```java
   /**
     * @param directory 缓存写入的目录，一般写在Sd卡的缓存目录，传入getExternalCacheDir()，在app卸载时会清空
     * @param appVersion app的版本，对应versionCode
     * @param valueCount 每条缓存对应的值的数量
     * @param maxSize 总缓存的大小，单位byte
     */
    public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
            throws IOException {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        if (valueCount <= 0) {
            throw new IllegalArgumentException("valueCount <= 0");
        }

        // prefer to pick up where we left off
        DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
        if (cache.journalFile.exists()) {
            try {
                cache.readJournal();
                cache.processJournal();
                cache.journalWriter = new BufferedWriter(new FileWriter(cache.journalFile, true),
                        IO_BUFFER_SIZE);
                return cache;
            } catch (IOException journalIsCorrupt) {
                cache.delete();
            }
        }

        // create a new empty cache
        directory.mkdirs();
        cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
        cache.rebuildJournal();
        return cache;
}
```

`open`方法创建了DiskLurCache实例，如果不存在journal文件，则创建并写入journal头，调用`rebuildJournal()`方法

```java
private synchronized void rebuildJournal() throws IOException {
        if (journalWriter != null) {
            journalWriter.close();
        }

        Writer writer = new BufferedWriter(new FileWriter(journalFileTmp), IO_BUFFER_SIZE);
        writer.write(MAGIC);
        writer.write("\n");
        writer.write(VERSION_1);
        writer.write("\n");
        writer.write(Integer.toString(appVersion));
        writer.write("\n");
        writer.write(Integer.toString(valueCount));
        writer.write("\n");
        writer.write("\n");

        for (Entry entry : lruEntries.values()) {
            if (entry.currentEditor != null) {
                writer.write(DIRTY + ' ' + entry.key + '\n');
            } else {
                writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
            }
        }

        writer.close();
        journalFileTmp.renameTo(journalFile);
        journalWriter = new BufferedWriter(new FileWriter(journalFile, true), IO_BUFFER_SIZE);
}
```

如果已经存在journal文件，则调用`readJournal()`和`processJournal()`对已经存在的缓存数据进行初始化，核心部分如下，

```java
  private void readJournal() throws IOException {
        InputStream in = new BufferedInputStream(new FileInputStream(journalFile), IO_BUFFER_SIZE);
        try {
		//验证journal文件头
		...

            while (true) {
                try {
				// 按行读取每一条记录
                    readJournalLine(readAsciiLine(in));
                } catch (EOFException endOfJournal) {
                    break;
                }
            }
        } finally {
            closeQuietly(in);
        }
}
```

`readJournalLine()`将每一行记录解析后添加到lruEntries,lruEntries是一个全局的LinkedHashMap<String, Entry>，用来保存所有的缓存对象Entry，排序方式以访问频次为权重，从少到多（*from least-recently accessed to most-recently accessed*）。Entry类包含了缓存的一些属性，如下：

```java
 private final class Entry {
        private final String key;

        /** Lengths of this entry's files. */
        private final long[] lengths;

        /** True if this entry has ever been published */
        private boolean readable;

        /** The ongoing edit or null if this entry is not being edited. */
        private Editor currentEditor;

        /** The sequence number of the most recently committed edit to this entry. */
        private long sequenceNumber;
}
```

`readJournalLine()`

```java
private void readJournalLine(String line) throws IOException {
        String[] parts = line.split(" ");
        //line like 'status key value[0].length ... value[N-1].length'
        //parts[0]=status,parts[1]=key
        if (parts.length < 2) {
            throw new IOException("unexpected journal line: " + line);
        }

        String key = parts[1];
        //从缓存列表中删除此条目
        if (parts[0].equals(REMOVE) && parts.length == 2) {
            lruEntries.remove(key);
            return;
        }

        Entry entry = lruEntries.get(key);
        if (entry == null) {
            entry = new Entry(key);
            lruEntries.put(key, entry);
        }

        if (parts[0].equals(CLEAN) && parts.length == 2 + valueCount) {
            entry.readable = true;
            entry.currentEditor = null;
            //数组下标从2到最后代表对应value的长度
            entry.setLengths(copyOfRange(parts, 2, parts.length));
        } else if (parts[0].equals(DIRTY) && parts.length == 2) {
            entry.currentEditor = new Editor(entry);
        } else if (parts[0].equals(READ) && parts.length == 2) {
            // this work was already done by calling lruEntries.get()
        } else {
            throw new IOException("unexpected journal line: " + line);
        }
}
```

`processJournal()`方法对lruEntries中的数据进行加工，删除脏数据，计算可用缓存数据的总长度，代码如下

```java
private void processJournal() throws IOException {
        deleteIfExists(journalFileTmp);//删除journal临时文件'journal.tmp'
        for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
            Entry entry = i.next();
            //没有Editor的数据表示可用的数据
            if (entry.currentEditor == null) {
                for (int t = 0; t < valueCount; t++) {
                    size += entry.lengths[t];
                }
            } else {
                //删除脏数据
                entry.currentEditor = null;
                for (int t = 0; t < valueCount; t++) {
                    deleteIfExists(entry.getCleanFile(t)); //删除 'key.index'
                    deleteIfExists(entry.getDirtyFile(t)); //删除 'key.index.tmp'
                }
                i.remove();
            }
        }
}
```

## 写入缓存

缓存的写入需要通过Editor类,调用`edit(String key)`获取此key对应的Editor对象。

```java
private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
        checkNotClosed(); //检测journalWriter是否为空
        validateKey(key); //key不能包含空格，'\r','\n'
        Entry entry = lruEntries.get(key);
        if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER
                && (entry == null || entry.sequenceNumber != expectedSequenceNumber)) {
            return null; // snapshot is stale
        }
        if (entry == null) {
            entry = new Entry(key);
            lruEntries.put(key, entry);
        } else if (entry.currentEditor != null) {
            return null; // another edit is in progress
        }

        Editor editor = new Editor(entry);
        entry.currentEditor = editor;

        // flush the journal before creating files to prevent file leaks
        //记录一个dirty日志
        journalWriter.write(DIRTY + ' ' + key + '\n');
        journalWriter.flush();
        return editor;
}
```
Editor类结构如下：
![](http://7xtfm0.com2.z0.glb.clouddn.com/editor.png)

- `newOutputStream()`方法获取一个输出流，将需要缓存的数据写入此OutputStream
- `newInputStream(int index)`方法获取对应index的输入流，对应最后一次写入的数据
- `getString(index)`是将`newInputStream(int index)`方法的返回值转化为String返回
- `set(int index, String value)`将value的值写入index对应的缓存中
- `commit()`提交此次操作
- `abort()`放弃此次操作

每次操作完成后，必须要调用`commit()`或者`abort()`,`commit()`和`abort()`方法内部都调用了DiskLruCache的`completeEdit(Editor editor, boolean success)`来通知DiskLruCache完成了一次edit操作，`completeEdit(Editor editor, boolean success)`方法内部对当前缓存所占用的内存进行检查，来决定是否需要释放一部分缓存，另外对journal日志文件的行数进行检查，来决定是否需要重写journal,代码如下

```java
    private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
        Entry entry = editor.entry;
        if (entry.currentEditor != editor) {
            throw new IllegalStateException();
        }

        // if this edit is creating the entry for the first time, every index must have a value
        if (success && !entry.readable) {
            for (int i = 0; i < valueCount; i++) {
                if (!entry.getDirtyFile(i).exists()) {
                    editor.abort();
                    throw new IllegalStateException("edit didn't create file " + i);
                }
            }
        }

        for (int i = 0; i < valueCount; i++) {
            File dirty = entry.getDirtyFile(i);
            if (success) {
                if (dirty.exists()) {
                    File clean = entry.getCleanFile(i);
                    dirty.renameTo(clean);
                    long oldLength = entry.lengths[i];
                    long newLength = clean.length();
                    entry.lengths[i] = newLength;
                    size = size - oldLength + newLength;
                }
            } else {
                deleteIfExists(dirty);
            }
        }

        redundantOpCount++;//记录操作的记录数
        entry.currentEditor = null;
        if (entry.readable | success) {
            entry.readable = true;
            journalWriter.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
            if (success) {
                //scquenceNumber用来判断缓存是否过期，每一次写入都会生成一个sequenceNumber到entry中
                //获取到的Snapshot对象中也有一个sequenceNumber，如果snapshot.sequenceNumber != entry.sequenceNumber
                //则代表获取到的Snapshot是过时的
                entry.sequenceNumber = nextSequenceNumber++;
            }
        } else {
            lruEntries.remove(entry.key);
            journalWriter.write(REMOVE + ' ' + entry.key + '\n');
        }

        //缓存总大小大于设置的缓存上限
        //日志记录大于2000条并且日志记录大于当前lruEntries.size();
        if (size > maxSize || journalRebuildRequired()) {
            executorService.submit(cleanupCallable);
        }
}
```

`journalRebuildRequired()`方法在journal文件日志行数大于2000并且日志行数大于lruEntries的大小时返回true。

`cleanupCallable`代码如下，主要对lruEnties的大小进行优化，删除最近使用次数较少的条目

```java
private final Callable<Void> cleanupCallable = new Callable<Void>() {
        @Override public Void call() throws Exception {
            synchronized (DiskLruCache.this) {
                if (journalWriter == null) {
                    return null; // closed
                }
                trimToSize();
                if (journalRebuildRequired()) {
                    rebuildJournal();
                    redundantOpCount = 0;
                }
            }
            return null;
        }
};
```

`trimToSize()`方法将lruEntries中使用较少的条目删除，直到lruEntries对应的缓存总大小小于预设的上限

```java
    private final Callable<Void> cleanupCallable = new Callable<Void>() {
        @Override public Void call() throws Exception {
            synchronized (DiskLruCache.this) {
                if (journalWriter == null) {
                    return null; // closed
                }
                trimToSize();
                if (journalRebuildRequired()) {
                    rebuildJournal();
                    redundantOpCount = 0;
                }
            }
            return null;
        }
};
```

## 读取缓存

读取缓存通过调用`get(String key)`方法，返回一个Snapshot对象

```java
  public synchronized Snapshot get(String key) throws IOException {
        checkNotClosed();
        validateKey(key);
        Entry entry = lruEntries.get(key);
        if (entry == null) {
            return null;
        }

        if (!entry.readable) {
            return null;
        }

        /*
         * Open all streams eagerly to guarantee that we see a single published
         * snapshot. If we opened streams lazily then the streams could come
         * from different edits.
         */
        InputStream[] ins = new InputStream[valueCount];
        try {
            for (int i = 0; i < valueCount; i++) {
                ins[i] = new FileInputStream(entry.getCleanFile(i));
            }
        } catch (FileNotFoundException e) {
            // a file must have been deleted manually!
            return null;
        }

        redundantOpCount++;
        journalWriter.append(READ + ' ' + key + '\n');
        if (journalRebuildRequired()) {
            executorService.submit(cleanupCallable);
        }

        return new Snapshot(key, entry.sequenceNumber, ins);
}
```

`get(String key)`方法内部也会检查是否需要重写journal，是否需要释放内存

Snapshot类结构如下：
![](http://7xtfm0.com2.z0.glb.clouddn.com/snapshot.png)

- `edit()`获取对应缓存的Editor
- `getInputStream(int index)`获取缓存的输出流
- `getString(int index)` 以String的方式返回缓存的值

## 移除缓存

通过`remove(String key)`方法移除key对应的缓存,`remove(String key)`方法也会对内存和journal进行审查，如下

```java
  public synchronized boolean remove(String key) throws IOException {
        checkNotClosed();
        validateKey(key);
        Entry entry = lruEntries.get(key);
        if (entry == null || entry.currentEditor != null) {
            return false;
        }

        for (int i = 0; i < valueCount; i++) {
            File file = entry.getCleanFile(i);
            if (!file.delete()) {
                throw new IOException("failed to delete " + file);
            }
            size -= entry.lengths[i];
            entry.lengths[i] = 0;
        }

        redundantOpCount++;
        journalWriter.append(REMOVE + ' ' + key + '\n');
        lruEntries.remove(key);

        if (journalRebuildRequired()) {
            executorService.submit(cleanupCallable);
        }

        return true;
}
```

## 关闭缓存

通过`close()`方法来关闭缓存

```java
    public synchronized void close() throws IOException {
        if (journalWriter == null) {
            return; // already closed
        }
        for (Entry entry : new ArrayList<Entry>(lruEntries.values())) {
            if (entry.currentEditor != null) {
                entry.currentEditor.abort();
            }
        }
        trimToSize();
        journalWriter.close();
        journalWriter = null;
}
```

## 其他API

- `deleteContents(File dir)` 删除dir或dir目录下所有文件
- `getDirectory()` 获取缓存的目录
- `maxSize()` 获取缓存上限
- `size()` 当前缓存占用内存总大小
- `isClosed()` 判断缓存是否关闭
- `delete()` 删除所有缓存
- `flush()` 将日志同步到journal文件中

### 参考

> [Caching Bitmaps](http://developer.android.com/training/displaying-bitmaps/cache-bitmap.html)  
> [Android DiskLruCache完全解析，硬盘缓存的最佳方案](http://blog.csdn.net/guolin_blog/article/details/28863651)  
> [DiskLruCache.java](https://android.googlesource.com/platform/libcore/+/android-4.1.1_r1/luni/src/main/java/  libcore/io/DiskLruCache.java)














