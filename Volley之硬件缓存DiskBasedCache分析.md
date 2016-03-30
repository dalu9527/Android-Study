# Volley之硬件缓存DiskBasedCache分析 #

Volley想必大家已经都不陌生了，2013年Google官方推荐的网络框架，经过这几年的发展，虽然被OkHttp、Retrofit各种“秒杀”，但是作为一款经典的网络框架库，还是有必要学习其实现过程，今天就来了解硬件缓存相关的文件，看看volley是如何处理缓存的。

## When：什么时候用到了缓存 ##

用过Volley的，都知道如何使用Volley，经典的三步曲就是：
1.创建一个RequestQueue对象

	RequestQueue mQueue = Volley.newRequestQueue(context); 

2.创建一个XXRequest对象

	XXRequest xxRequest = new XXRequest("http://www.baidu.com",
						new Response.Listener<XX>() {
							@Override
							public void onResponse(XX response) {
								Log.d("TAG", response);
							}
						}, new Response.ErrorListener() {
							@Override
							public void onErrorResponse(VolleyError error) {
								Log.e("TAG", error.getMessage(), error);
							}
						});
	

3.将StringRequest对象添加到RequestQueue里面

	mQueue.add(stringRequest);

当我们在第一步初始化RequestQueue时，缓存就已经初始化了，看看源代码：
    
	public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();

        return queue;
    }

关键的代码就在

	RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);

初始化RequestQueue的时候就已经初始化了DiskBasedCache--今天要分析的主角。

## How： DiskBasedCache如何实现的##

看源代码，发现：

    public class DiskBasedCache implements Cache

其是实现了CaChe这个缓存类，这个缓存类的源码如下：

    public interface Cache {
    
    public Entry get(String key);//获取缓存数据
    
    public void put(String key, Entry entry);//存储缓存数据

    public void initialize();//初始化本地缓存数据

    public void invalidate(String key, boolean fullExpire);//缓存失效

    public void remove(String key);//移除指定Key对应的缓存

    public void clear();//清楚缓存

    public static class Entry {//缓存的数据结构
        /** The data returned from cache. */
        public byte[] data;//缓存数据

        /** ETag for cache coherency. */
        public String etag;//缓存标志，和Http缓存对应

        /** Date of this response as reported by the server. */
        public long serverDate;//响应日期 

        /** The last modified date for the requested object. */
        public long lastModified;//最后修改日期

        /** TTL for this record. */
        public long ttl;//Time To Live 生存时间

        /** Soft TTL for this record. */
        public long softTtl;//软生存时间

        /** Immutable response headers as received from server; must be non-null. */
        public Map<String, String> responseHeaders = Collections.emptyMap();//响应头，必须为非空

        /** True if the entry is expired. */
        public boolean isExpired() {//是否超时
            return this.ttl < System.currentTimeMillis();
        }

        /** True if a refresh is needed from the original data source. */
        public boolean refreshNeeded() {//缓存是否需要更新
            return this.softTtl < System.currentTimeMillis();
        }
    }
	｝


CaChe作为接口，定义了缓存数据Entry的数据结构类型（一个Entry对应一条缓存）、定义了缓存的基本操作，等待子类实现。

在Entry的定义中，存在TTL和SoftTTl，看代码的时候一直对这两个不明白，在CacheDispatcher文件中的run方法中，可以看到如下的代码：

    if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }

而refreshNeeded()方法如下：


    /** True if a refresh is needed from the original data source. */
        public boolean refreshNeeded() {
            return this.softTtl < System.currentTimeMillis();
        }

可以看出，ttl和softtl的区别：ttl小于当前时间，则直接过期；当softTtl小于当前时间，需要做刷新操作。


## What： DiskBasedCache干了什么##


根据接口CaChe的定义，我们可以知道DiskBasedCache必定会实现其方法，下面就看看其内部结构。

### 构造方法： ###
  
    public DiskBasedCache(File rootDirectory, int maxCacheSizeInBytes) {
        mRootDirectory = rootDirectory;
        mMaxCacheSizeInBytes = maxCacheSizeInBytes;
    }

    /**
     * Constructs an instance of the DiskBasedCache at the specified directory using
     * the default maximum cache size of 5MB.
     * @param rootDirectory The root directory of the cache.
     */
    public DiskBasedCache(File rootDirectory) {
        this(rootDirectory, DEFAULT_DISK_USAGE_BYTES);
    }

构造函数中，可以发现，默认的缓存大小是5M


### 实现CaChe的方法：put ###

上代码：

    public synchronized void put(String key, Entry entry) {
        pruneIfNeeded(entry.data.length);//修改当前缓存大小使之适应最大缓存大小
        File file = getFileForKey(key);
        try {
            BufferedOutputStream fos = new BufferedOutputStream(new FileOutputStream(file));
            CacheHeader e = new CacheHeader(key, entry);//缓存头,保存缓存的信息在内存
            boolean success = e.writeHeader(fos);//写入缓存头
            if (!success) {
                fos.close();
                VolleyLog.d("Failed to write header for %s", file.getAbsolutePath());
                throw new IOException();
            }
            fos.write(entry.data);
            fos.close();
            putEntry(key, e);
            return;
        } catch (IOException e) {
        }
        boolean deleted = file.delete();
        if (!deleted) {
            VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
        }
    }

整个存储过程为：

1. 检查要缓存的数据的长度，如果当前已经缓存的数据大小mTotalSize + 要缓存的数据大小，大于缓存最大值mMaxCacheSizeInBytes，则要将旧的缓存文件删除，以腾出空间来存储新的缓存文件 --由pruneIfNeeded()解决；
2. 根据缓存记录类Entry，提取Entry除了数据以外的其他信息，例如这个缓存的大小，过期时间，写入日期等，并且将这些信息实例成CacheHeader。这样做的目的是，方便以后我们查询缓存，获得缓存相应信息时，不需要去读取硬盘，因为CacheHeader是内存中的。
3. 写入缓存

根据整个过程，一个个的分析：

首先是由pruneIfNeeded()方法：不断删除文件，直到腾出足够的空间给新的缓存文件

    private void pruneIfNeeded(int neededSpace) {
        if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes) {
            return;//如果没有超过最大缓存大小,返回
        }
        if (VolleyLog.DEBUG) {
            VolleyLog.v("Pruning old cache entries.");
        }

        long before = mTotalSize;
        int prunedFiles = 0;
        long startTime = SystemClock.elapsedRealtime();

        Iterator<Map.Entry<String, CacheHeader>> iterator = mEntries.entrySet().iterator();
        while (iterator.hasNext()) {//遍历缓存文件信息
            Map.Entry<String, CacheHeader> entry = iterator.next();
            CacheHeader e = entry.getValue();
            boolean deleted = getFileForKey(e.key).delete();//删除文件
            if (deleted) {
                mTotalSize -= e.size;//更新文件大小
            } else {
               VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",
                       e.key, getFilenameForKey(e.key));
            }
            iterator.remove();
            prunedFiles++;

            if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes * HYSTERESIS_FACTOR) {//HYSTERESIS_FACTOR = 0.9f;一直到新对象添加进来后还会有10%的空间剩余时为止
                break;
            }
        }

        if (VolleyLog.DEBUG) {
            VolleyLog.v("pruned %d files, %d bytes, %d ms",
                    prunedFiles, (mTotalSize - before), SystemClock.elapsedRealtime() - startTime);
        }
    }

这个函数中，涉及到了一个数据结构：mEntries，其定义为：

    private final Map<String, CacheHeader> mEntries =
            new LinkedHashMap<String, CacheHeader>(16, .75f, true);

其定义了一个final的LinkedHashMap（初始大小为16、装载因子为0.75、排序为访问顺序），该map提供了内存缓存的能力，并且将map内的记录安装访问顺序排序，而起内部排序方法是LRU算法，每一次存储缓存，都会修改这个map，也就是说要调用LRU算法进行重新排序。

另外，在删减缓存头部CacheHeader 过程中，有一个细节：是满足还有剩余10%的空间就认为已经删减出足够的空间，而不是剩余0%的空间（我想可能是开发人员基于空间、效率的一个折算，当然我们可以自己修改它）

下面我们看看刚才提到的CacheHeader，其定义如下：

    static class CacheHeader {
        /** The size of the data identified by this CacheHeader. (This is not
         * serialized to disk. */
        public long size;//缓存数据大小 

        /** The key that identifies the cache entry. */
        public String key;//缓存键值 

        /** ETag for cache coherence. */
        public String etag;//etag 是HTTP头部的一个定义，是HTTP协议提供的若干机制中的一种Web缓存验证机制

        /** Date of this response as reported by the server. */
        public long serverDate;//保存日期 

        /** The last modified date for the requested object. */
        public long lastModified;//上次修改时间 

        /** TTL for this record. */
        public long ttl;//生存时间 

        /** Soft TTL for this record. */
        public long softTtl;//软生存时间

        /** Headers from the response resulting in this cache entry. */
        public Map<String, String> responseHeaders;//响应头 

        private CacheHeader() { }

        /**
         * Instantiates a new CacheHeader object
         * @param key The key that identifies the cache entry
         * @param entry The cache entry.
         */
        public CacheHeader(String key, Entry entry) {
            this.key = key;
            this.size = entry.data.length;
            this.etag = entry.etag;
            this.serverDate = entry.serverDate;
            this.lastModified = entry.lastModified;
            this.ttl = entry.ttl;
            this.softTtl = entry.softTtl;
            this.responseHeaders = entry.responseHeaders;
        }

        /**
         * Reads the header off of an InputStream and returns a CacheHeader object.
         * @param is The InputStream to read from.
         * @throws IOException
         */
        public static CacheHeader readHeader(InputStream is) throws IOException {//读取缓存头
            CacheHeader entry = new CacheHeader();
            int magic = readInt(is);
            if (magic != CACHE_MAGIC) {
                // don't bother deleting, it'll get pruned eventually
                throw new IOException();
            }
            entry.key = readString(is);
            entry.etag = readString(is);
            if (entry.etag.equals("")) {
                entry.etag = null;
            }
            entry.serverDate = readLong(is);
            entry.lastModified = readLong(is);
            entry.ttl = readLong(is);
            entry.softTtl = readLong(is);
            entry.responseHeaders = readStringStringMap(is);

            return entry;
        }

        /**
         * Creates a cache entry for the specified data.
         */
        public Entry toCacheEntry(byte[] data) {//转换
            Entry e = new Entry();
            e.data = data;
            e.etag = etag;
            e.serverDate = serverDate;
            e.lastModified = lastModified;
            e.ttl = ttl;
            e.softTtl = softTtl;
            e.responseHeaders = responseHeaders;
            return e;
        }


        /**
         * Writes the contents of this CacheHeader to the specified OutputStream.
         */
        public boolean writeHeader(OutputStream os) {//写入缓存头
            try {
                writeInt(os, CACHE_MAGIC);
                writeString(os, key);
                writeString(os, etag == null ? "" : etag);
                writeLong(os, serverDate);
                writeLong(os, lastModified);
                writeLong(os, ttl);
                writeLong(os, softTtl);
                writeStringStringMap(responseHeaders, os);
                os.flush();
                return true;
            } catch (IOException e) {
                VolleyLog.d("%s", e.toString());
                return false;
            }
        }

    }

通过构造函数可知，其就是把Entry类里面的除了data以外的信息提取出来而已

了解以上信息后，回到put方法内，腾出足够的空间后，就是将缓存信息写入到了CacheHeader中，完成缓存写入的过程。

### 实现CaChe的方法：get ###

写入缓存，就肯定存在读取缓存的方法：

    public synchronized Entry get(String key) {
        CacheHeader entry = mEntries.get(key);
        // if the entry does not exist, return.
        if (entry == null) {
            return null;
        }

        File file = getFileForKey(key);//获取缓存文件
        CountingInputStream cis = null;
        try {
            cis = new CountingInputStream(new BufferedInputStream(new FileInputStream(file)));
            CacheHeader.readHeader(cis); // eat header读取头部
            byte[] data = streamToBytes(cis, (int) (file.length() - cis.bytesRead));
            return entry.toCacheEntry(data);
        } catch (IOException e) {
            VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
            remove(key);
            return null;
        } finally {
            if (cis != null) {
                try {
                    cis.close();
                } catch (IOException ioe) {
                    return null;
                }
            }
        }
    }

读取过程很简单

1. 读取缓存文件数据
2. 读取缓存文件头部 CacheHeader
3. 生成Entry,返回

在方法中涉及到了getFileForKey()方法：

    public File getFileForKey(String key) {
        return new File(mRootDirectory, getFilenameForKey(key));
    }

继续看getFilenameForKey()：根据key返回文件名

    private String getFilenameForKey(String key) {
        int firstHalfLength = key.length() / 2;
        String localFilename = String.valueOf(key.substring(0, firstHalfLength).hashCode());
        localFilename += String.valueOf(key.substring(firstHalfLength).hashCode());
        return localFilename;
    }

这个就有意思了：使用Key的前半段的hashCode和后半段的hashCode连接起来，这个使用就非常的耐人寻味了。

#### [Why：使用两次hashcode](http://stackoverflow.com/questions/34984302/why-volley-diskbasedcache-splicing-without-direct-access-to-the-cache-file-name/34987423#34987423) ####

之前看代码的时候，看到这里，发现用了两次hashcode，纳闷为什么不是直接返回key.hashCode();
看一下hashCode方法：

    public int hashCode() {
        int hash = hashCode;
        if (hash == 0) {
            if (count == 0) {
                return 0;
            }
            final int end = count + offset;
            final char[] chars = value;
            for (int i = offset; i < end; ++i) {
                hash = 31*hash + chars[i];
            }
            hashCode = hash;
        }
        return hash;
    }

其是利用 hash = 31*hash + chars[i];进行的，而hashCode并不能保证每次都是唯一的，例如[hashCode可能会相同](http://my.oschina.net/backtract/blog/169310?fromerr=xYxmlOMX)

而Volley在获取文件名的时候采用两次hash，目的估计也是为了尽可能避免hashcode重复从而导致误认为文件名重复。

### 实现CaChe的方法：initialize ###

使用Volley时，需要先计算本地缓存：

    public synchronized void initialize() {
        if (!mRootDirectory.exists()) {
            if (!mRootDirectory.mkdirs()) {
                VolleyLog.e("Unable to create cache dir %s", mRootDirectory.getAbsolutePath());
            }
            return;
        }

        File[] files = mRootDirectory.listFiles();
        if (files == null) {
            return;
        }
        for (File file : files) {
            BufferedInputStream fis = null;
            try {
                fis = new BufferedInputStream(new FileInputStream(file));
                CacheHeader entry = CacheHeader.readHeader(fis);
                entry.size = file.length();
                putEntry(entry.key, entry);
            } catch (IOException e) {
                if (file != null) {
                   file.delete();
                }
            } finally {
                try {
                    if (fis != null) {
                        fis.close();
                    }
                } catch (IOException ignored) { }
            }
        }
    }

计算空间的方法就是putEntry():

    private void putEntry(String key, CacheHeader entry) {
        if (!mEntries.containsKey(key)) {
            mTotalSize += entry.size;
        } else {
            CacheHeader oldEntry = mEntries.get(key);
            mTotalSize += (entry.size - oldEntry.size);
        }
        mEntries.put(key, entry);
    }

并且会将缓存放入到mEntries中。

### 实现CaChe的方法：invalidate ###

    public synchronized void invalidate(String key, boolean fullExpire) {
        Entry entry = get(key);
        if (entry != null) {
            entry.softTtl = 0;
            if (fullExpire) {
                entry.ttl = 0;
            }
            put(key, entry);
        }

    }

 该方法 将key对应的缓存作废 ；如果fullExpire为true，则将整个entry作废 ；如果为false,则只是软作废，也就是将缓存置于需要刷新的状态。

### 实现CaChe的方法：clear ###
    public synchronized void clear() {
        File[] files = mRootDirectory.listFiles();
        if (files != null) {
            for (File file : files) {
                file.delete();
            }
        }
        mEntries.clear();
        mTotalSize = 0;
        VolleyLog.d("Cache cleared.");
    }

这个嘛，一看大家就知道其用途了

### 实现CaChe的方法：remove ###

    public synchronized void remove(String key) {
        boolean deleted = getFileForKey(key).delete();
        removeEntry(key);
        if (!deleted) {
            VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",
                    key, getFilenameForKey(key));
        }
    }

removeEntry():

    private void removeEntry(String key) {
        CacheHeader entry = mEntries.get(key);
        if (entry != null) {
            mTotalSize -= entry.size;
            mEntries.remove(key);
        }
    }

嗯，这个，都懂得

## 总结 ##

至此，DiskBasedCache方法基本已经掌握的差不多，Volley自身的缓存策略还是值得我们深入研究的，一些处理的细节也是非常的耐人寻味，需要我们慢慢的“品位”。Volley在使用缓存时，会首先判断缓存是否存在，如果不存在则访问网络，并且数据获得后第一时间写入缓存中。当然我们可以自己扩展缓存，基于CaChe类，实现更强大的缓存。

作为扩展，可能还需要我们了解一下Entry.etag的作用，我们知道Volley的缓存是基于[Http协议](http://blog.csdn.net/dxyoo7/article/details/27200947)的，大家可以继续了解协议相关的缓存策略，再理解Volley的源码就更轻松了。