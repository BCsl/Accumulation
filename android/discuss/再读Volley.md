## 再读Volley
第一次看Volley的代码的时候只是大概地理清了它的结构而没有做细节上的记录，但是看了这两篇文章之后，[面试后的总结](http://kymjs.com/code/2016/03/08/01/)和[Android网络请求心路历程](http://www.jianshu.com/p/3141d4e46240)，发现Volley还有很多可以学习的地方，所以再次研读

## HTTP缓存机制
下面的头信息中涉及到的时间的字段都是RFC1123格式，Android开发可以用这方法来解析`org.apache.http.impl.cookie.DateUtils.parseDate(dateStr).getTime()`

### 相关的Request字段
|   请求头字段 |    意义 |
|-------------|---------|
| If-None-Match: "737060cd8c284d8af7ad3082f209582d" | 缓存文件的Etag(Hash)值，与服务器回应的Etag比较判断是否改变 |
| If-Modified-Since: Sat, 29 Oct 2010 19:43:31 GMT | 缓存文件的最后修改时间 |

### 相关的Response字段
|   响应字段 |    意义 |
|-------------|---------|
| Cache-Control:  max-age=600, stale-while-revalidate=30 | 告诉所有的缓存机制是否可以缓存及哪种类型 |
| Expires: Thu, 01 Dec 2010 16:00:00 GMT | 响应过期的日期和时间 |
| Last-Modified: Tue, 15 Nov 2010 12:45:26 GMT | 请求资源的最后修改时间 |
| ETag: "737060cd8c284d8af7ad3082f209582d" | 请求变量的实体标签的当前值 |

### HTTP 304状态分析
所请求的资源未修改，服务器返回此状态码时，不会返回任何资源。客户端通常会缓存访问过的资源，通过提供一个头信息指出客户端希望只返回在指定日期之后修改的资源（（一般是提供If-Modified-Since头表示客户只想比指定日期更新的文档）

### Volley是如何处理Http缓存机制

#### 请求返回阶段

在`parseNetworkResponse`方法，需要用户把`NetworkResponse`对象转换成`Response<T>`对象，除了目标的结果类型`T`外，还需要一个`Cache.Entity`类型的参数来构造一个`Response`对象，一般只需要使用Volley提供的Api来生成`Cache.Entity`，如下是lib提供的StringRequest的`parseNetworkResponse`方法实现：
```java
@Override
protected Response<String> parseNetworkResponse(NetworkResponse response) {
    String parsed;
    try {
        parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
    } catch (UnsupportedEncodingException e) {
        parsed = new String(response.data);
    }
    return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
}
```
其中的`HttpHeaderParser.parseCacheHeaders(response)`方法用来构造`Cache.Entity`对象，下面是它的实现，这个过程计算并记录了两个时间，`softExpire`和`finalExpire`，分别控制的是缓存需要刷新的时间和最终过期的时间：

```java
public static Cache.Entry parseCacheHeaders(NetworkResponse response) {
    long now = System.currentTimeMillis();

    Map<String, String> headers = response.headers;

    long serverDate = 0;
    long lastModified = 0;
    long serverExpires = 0;
    long softExpire = 0;
    long finalExpire = 0;
    long maxAge = 0;
    long staleWhileRevalidate = 0;
    boolean hasCacheControl = false;

    String serverEtag = null;
    String headerValue;

    headerValue = headers.get("Date");//请求发送的日期和时间
    if (headerValue != null) {
        serverDate = parseDateAsEpoch(headerValue);
    }
    //对于这样的Cache-Control:  max-age=600, stale-while-revalidate=30，指示着在600秒内数据是最新的，
    headerValue = headers.get("Cache-Control");//支持的缓存类型
    if (headerValue != null) {
        hasCacheControl = true;
        String[] tokens = headerValue.split(",");
        for (int i = 0; i < tokens.length; i++) {
            String token = tokens[i].trim();
            if (token.equals("no-cache") || token.equals("no-store")) {
                return null;    //no-cache和no-store代表服务器五支持缓存，直接返回NULL
            } else if (token.startsWith("max-age=")) {
                try {
                    maxAge = Long.parseLong(token.substring(8));
                } catch (Exception e) {
                }
            } else if (token.startsWith("stale-while-revalidate=")) {
                try {
                    staleWhileRevalidate = Long.parseLong(token.substring(23));
                } catch (Exception e) {
                }
            } else if (token.equals("must-revalidate") || token.equals("proxy-revalidate")) {
                maxAge = 0;
            }
        }
    }

    headerValue = headers.get("Expires");//响应过期的日期和时间
    if (headerValue != null) {
        serverExpires = parseDateAsEpoch(headerValue);
    }

    headerValue = headers.get("Last-Modified");//请求资源的最后修改时间
    if (headerValue != null) {
        lastModified = parseDateAsEpoch(headerValue);
    }

    serverEtag = headers.get("ETag");// 请求变量的实体标签的当前值

    // Cache-Control takes precedence over an Expires header, even if both exist and Expires
    // is more restrictive.
    if (hasCacheControl) {//是否有Cache-Control头，且非no-cache和no-store方式
        softExpire = now + maxAge * 1000;
        finalExpire = softExpire + staleWhileRevalidate * 1000;
    } else if (serverDate > 0 && serverExpires >= serverDate) {
        // Default semantic for Expire header in HTTP specification is softExpire.
        softExpire = now + (serverExpires - serverDate);
        finalExpire = softExpire;
    }

    Cache.Entry entry = new Cache.Entry();
    entry.data = response.data;
    entry.etag = serverEtag;
    entry.softTtl = softExpire;
    entry.ttl = finalExpire;//time to live
    entry.serverDate = serverDate;
    entry.lastModified = lastModified;
    entry.responseHeaders = headers;

    return entry;
}

```

解析成`Response`对象之后，就是把需要缓存的实体放入缓存:

NetworkDispatcher.java
```java
Response<?> response = request.parseNetworkResponse(networkResponse);
request.addMarker("network-parse-complete");

// Write to cache if applicable.
// TODO: Only update cache metadata instead of entire record for 304s.
if (request.shouldCache() && response.cacheEntry != null) {
    mCache.put(request.getCacheKey(), response.cacheEntry);
    request.addMarker("network-cache-written");
}
```
#### 请求阶段

对于需要使用缓存的请求会想加入到缓存队列，由`CacheDispatcher`来分发处理，其处理过程，分了三种情况
* 没有缓存，那就把请求加入到网络请求队列
* 缓存过期，那就把请求加入到网络请求队列，还添加了必要的缓存信息，用于控制请求头信息
* 缓存需要更新了，先马上返回数据给用户让用户展示，然后接着就把请求加入到网络请求队列

```java
@Override
public void run() {
    if (DEBUG) VolleyLog.v("start new dispatcher");
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    // Make a blocking call to initialize the cache.
    mCache.initialize();

    while (true) {
        try {
            // Get a request from the cache triage queue, blocking until at least one is available.
            final Request<?> request = mCacheQueue.take();
            //...省略部分代码...
            // Attempt to retrieve this item from cache.
            Cache.Entry entry = mCache.get(request.getCacheKey());
            if (entry == null) {
                request.addMarker("cache-miss");
                // Cache miss; send off to the network dispatcher.
                mNetworkQueue.put(request);
                continue;
            }
            // If it is completely expired, just send it to the network.
            if (entry.isExpired()) {//判断依据:this.ttl < System.currentTimeMillis()
                request.addMarker("cache-hit-expired");
                //虽然失效了，但还是为请求设置了缓存实体，用来在请求的时候控制请求头信息
                request.setCacheEntry(entry);
                mNetworkQueue.put(request);
                continue;
            }
            // We have a cache hit; parse its data for delivery back to the request.
            request.addMarker("cache-hit");
            Response<?> response = request.parseNetworkResponse(
                    new NetworkResponse(entry.data, entry.responseHeaders));
            request.addMarker("cache-hit-parsed");

            if (!entry.refreshNeeded()) {//判断依据this.softTtl < System.currentTimeMillis()
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
        } catch (InterruptedException e) {
            // We may have been interrupted because it was time to quit.
            if (mQuit) {
                return;
            }
            continue;
        }
    }
}

```

从上面可知，当缓存过期或者需要更新的时候，会再次请求网络请求，这时候会通过记录的缓存实体的信息构造请求头，告诉服务器一些信息判断是否需要返回新的消息实体

##### 添加Cache相关头信息
```java
BasicNetwork.java

//这个是在请求可以从缓存查到，但是需要更新，构造和cache相关的header
private void addCacheHeaders(Map<String, String> headers, Cache.Entry entry) {
    // If there's no cache entry, we're done.
    if (entry == null) {
        return;
    }
    if (entry.etag != null) {
        headers.put("If-None-Match", entry.etag);
    }
    if (entry.lastModified > 0) {
        Date refTime = new Date(entry.lastModified);
        headers.put("If-Modified-Since", DateUtils.formatDate(refTime));
    }
}
```

##### 处理网络请求结果

这过程会根据`Cache.Entity`实体添加头信息，处理304返回码更新缓存头信息，还有重试策略

```java
@Override
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
    long requestStart = SystemClock.elapsedRealtime();
    while (true) {
        HttpResponse httpResponse = null;
        byte[] responseContents = null;
        Map<String, String> responseHeaders = Collections.emptyMap();
        try {
            // Gather headers.
            Map<String, String> headers = new HashMap<String, String>();
            //在请求可以从缓存查到，但是需要更新，构造和cache相关的header
            addCacheHeaders(headers, request.getCacheEntry());
            httpResponse = mHttpStack.performRequest(request, headers);
            StatusLine statusLine = httpResponse.getStatusLine();
            int statusCode = statusLine.getStatusCode();

            responseHeaders = convertHeaders(httpResponse.getAllHeaders());
            // Handle cache validation.304
            if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
                //NOT_MODIFIED,直接使用缓存
                Entry entry = request.getCacheEntry();
                if (entry == null) {
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                            responseHeaders, true,
                            SystemClock.elapsedRealtime() - requestStart);
                }

                // A HTTP 304 response does not have all header fields. We
                // have to use the header fields from the cache entry plus
                // the new ones from the response.
                // http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5
                entry.responseHeaders.putAll(responseHeaders);//更新头信息
                return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                        entry.responseHeaders, true,
                        SystemClock.elapsedRealtime() - requestStart);
            }

            // Some responses such as 204s do not have content.  We must check.
            if (httpResponse.getEntity() != null) {
              responseContents = entityToBytes(httpResponse.getEntity());
            } else {
              // Add 0 byte response as a way of honestly representing a
              // no-content request.
              responseContents = new byte[0];
            }

            // if the request is slow, log it.
            long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
            logSlowRequests(requestLifetime, request, responseContents, statusLine);

            if (statusCode < 200 || statusCode > 299) {
                throw new IOException();
            }
            return new NetworkResponse(statusCode, responseContents, responseHeaders, false,
                    SystemClock.elapsedRealtime() - requestStart);
        } catch (SocketTimeoutException e) {
            attemptRetryOnException("socket", request, new TimeoutError());
        } catch (ConnectTimeoutException e) {
            attemptRetryOnException("connection", request, new TimeoutError());
        } catch (MalformedURLException e) {
            throw new RuntimeException("Bad URL " + request.getUrl(), e);
        } catch (IOException e) {
            int statusCode = 0;
            NetworkResponse networkResponse = null;
            if (httpResponse != null) {
                statusCode = httpResponse.getStatusLine().getStatusCode();
            } else {
                throw new NoConnectionError(e);
            }
            VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
            if (responseContents != null) {
                networkResponse = new NetworkResponse(statusCode, responseContents,
                        responseHeaders, false, SystemClock.elapsedRealtime() - requestStart);
                if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                        statusCode == HttpStatus.SC_FORBIDDEN) {
                    attemptRetryOnException("auth",
                            request, new AuthFailureError(networkResponse));
                } else {
                    // TODO: Only throw ServerError for 5xx status codes.
                    throw new ServerError(networkResponse);
                }
            } else {
                throw new NetworkError(networkResponse);
            }
        }
    }
}

```

## 本地缓存的分析

默认的缓存实现使用`DiskBasedCache`类，需要在构造的时候指定缓存目录和缓存大小（默认5M），内部维护了一个`LinkedHashMap<String, CacheHeader>`类型的缓存表，在初始化的时候遍历读取缓存目录下所有文件，以一定的格式读取（静态方法`readHeader`）构造成`CacheHeader`对象，并保存在`LinkedHashMap`中，另外一些需要注意的是每次的`put`都是直接先写文件，然后再存到`LinkedHashMap`，`remove`则是删除文件，再移除出`LinkedHashMap`，所以尽量在非UI线程上操作

```java
public static CacheHeader readHeader(InputStream is) throws IOException {
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
    entry.ttl = readLong(is);
    entry.softTtl = readLong(is);
    entry.responseHeaders = readStringStringMap(is);

    try {
        entry.lastModified = readLong(is);
    } catch (EOFException e) {
        // the old cache entry format doesn't know lastModified
    }

    return entry;
}
```

### 如何控制缓存大小

这个应该是比较好做，因为存入的数据大小我们是知道的，而且当前缓存库总大小也是有记录的`mTotalSize`字段，所以每次加入缓存前，先做好判断是否需要移除部分近期最少使用的缓存信息（LinkedHashMap可以通过最后更新时间或者插入时间的顺序来遍历的）,实现，唯一有疑问的是为什么乘以`HYSTERESIS_FACTOR`，为了避免进行扩容？，暂时不得而知：
```java
private void pruneIfNeeded(int neededSpace) {
    if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes) {
        return;
    }
    long before = mTotalSize;
    int prunedFiles = 0;
    long startTime = SystemClock.elapsedRealtime();

    Iterator<Map.Entry<String, CacheHeader>> iterator = mEntries.entrySet().iterator();
    while (iterator.hasNext()) {
        Map.Entry<String, CacheHeader> entry = iterator.next();
        CacheHeader e = entry.getValue();
        boolean deleted = getFileForKey(e.key).delete();
        if (deleted) {
            mTotalSize -= e.size;
        } else {
           VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",e.key, getFilenameForKey(e.key));
        }
        iterator.remove();
        prunedFiles++;
        if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes * HYSTERESIS_FACTOR) {
            break;
        }
    }
}
```
### Volley缓存命中率的优化
上面的方式是通过`LRU`算法来控制缓存库大小的，如果让你去设计Volley的缓存功能，你要如何增大它的命中率？[面试后的总结](http://kymjs.com/code/2016/03/08/01/)也提出了一个很好的方案，先尝试去删除已经失效的缓存，但这可能需要两次遍历操作，是否有不妥？或者增加一个`TreeMap`来记录，先对`TreeMap`进行顺序输出来删除（如果遇到没过期就BREAK），按照过期时间来作为KEY（KEY的选择有问题），但是这个方案也不太好，虽然遍历上可能可以省部分时间，但需要同时删除两个`MAP`中的实体的时候，`KEY`对不上

### Volley缓存文件名的计算
[面试后的总结](http://kymjs.com/code/2016/03/08/01/)中提到是为了避免hash冲突，之前面试的时候也被面试官提问过类似问题，不太了解这方面的信息，只了解`HashMap`是通过分离链接法来处理的，先挖个坑
