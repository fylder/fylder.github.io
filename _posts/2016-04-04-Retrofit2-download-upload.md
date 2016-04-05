---
published: true
layout: post
title: Retrofit2的上传下载
category: android
tags: 
 - Retrofit2 
time: 2016.4.4 20:47:24
excerpt: 用拦截器在上传下载的监听
 
---

近段时间一直在看Retrofit2和RxJava，收集了不少相关资料，文件的上传下载有时候我们需要知道一个进度，用于给用户提供一个友好的交互提示，要实现这个功能，需要在过程中额外的加入统计，正好okhttp提供一个Interceptor拦截器。


### 下载

首先定义一个api，加上 `@Streaming` 是为了防止在大文件的情况的下可能会导致OOM，没有注入，在Monitors的Memory里看到内存一直增长，也就是说过程中一直持续保存在内存，直到下载完成后才会向内存读取写入，而加入后就会边接收边写入，完成合理的IO，内存就不会波动很大。

返回值就需要的一个ResponseBody，文件的写入就需要这个来获取。


**定义一个api**
```java
/**
 * beware with large files
 */
 @Streaming
 @GET("downlad")
 Observable<ResponseBody> download();
```

**如何保存文件**
只需要在map操作符里处理就可以
> .map(r -> writeResponseBodyToDisk(response, path, name))

```java
/**
 * IO写入sdcard
 */
private static boolean writeResponseBodyToDisk(ResponseBody response, String path, String name) {
    try {
        // todo change the file location/name according to your needs
        File futureStudioIconFile = new File(path, name);

        InputStream inputStream = null;
        OutputStream outputStream = null;

        try {
            byte[] fileReader = new byte[4096];

            long fileSize = response.contentLength();
            long fileSizeDownloaded = 0;

            inputStream = response.byteStream();
            outputStream = new FileOutputStream(futureStudioIconFile);

            while (true) {
                int read = inputStream.read(fileReader);

                if (read == -1) {
                    break;
                }
                outputStream.write(fileReader, 0, read);
                fileSizeDownloaded += read;
                Logger.t("writeResponseBodyToDisk").d("file download: " + fileSizeDownloaded + " of " + fileSize);
            }

            outputStream.flush();
            return true;
        } catch (IOException e) {
            Logger.t("IOException").d("file download: " + e.getMessage());
            return false;
        } finally {
            if (inputStream != null) {
                inputStream.close();
            }
            if (outputStream != null) {
                outputStream.close();
            }
        }
    } catch (IOException e) {
        return false;
    }
}
```

**下载拦截监听进度的接口**

```java
public interface DownloadListener {

    /**
     * @param bytesRead     已下载
     * @param contentLength 总大小
     * @param done          下载是否完成
     */
    void update(long bytesRead, long contentLength, boolean done);
    
}
```
**下载拦截器**
```java
public class DownloadInterceptor implements Interceptor {

    private DownloadListener progressListener;

    public DownloadInterceptor(DownloadListener progressListener) {
        this.progressListener = progressListener;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Response originalResponse = chain.proceed(chain.request());
        return originalResponse.newBuilder()
                .body(new DownloadProgressResponseBody(originalResponse.body(), progressListener))
                .build();
    }

    private static class DownloadProgressResponseBody extends ResponseBody {

        private final ResponseBody responseBody;
        private final DownloadListener progressListener;
        private BufferedSource bufferedSource;

        public DownloadProgressResponseBody(ResponseBody responseBody, DownloadListener progressListener) {
            this.responseBody = responseBody;
            this.progressListener = progressListener;
        }

        @Override
        public MediaType contentType() {
            return responseBody.contentType();
        }

        @Override
        public long contentLength() {
            return responseBody.contentLength();
        }

        @Override
        public BufferedSource source() {
            if (bufferedSource == null) {
                bufferedSource = Okio.buffer(source(responseBody.source()));
            }
            return bufferedSource;
        }

        private Source source(Source source) {
            return new ForwardingSource(source) {
                long totalBytesRead = 0L;

                @Override
                public long read(Buffer sink, long byteCount) throws IOException {
                    long bytesRead = super.read(sink, byteCount);
                    // read() returns the number of bytes read, or -1 if this source is exhausted.
                    totalBytesRead += bytesRead != -1 ? bytesRead : 0;

                    if (null != progressListener) {
                        progressListener.update(totalBytesRead, responseBody.contentLength(), bytesRead == -1);
                    }
                    return bytesRead;
                }
            };
        }
    }
}
```
**在获取一个Retrofit时在OkHttpClient加入拦截器就可以监听**
```java
OkHttpClient okHttpClient = new OkHttpClient()
        .newBuilder()
        .addInterceptor(new UploadInterceptor(listener))
        .build();
```

### 上传

Http的上传head需要定义Content-Type: multipart/form-data;
Retrofit2的在上传要注入`@Multipart`，参数就`@PartMap`或`@Part`

**定义api**
```java
@Multipart
@POST("upload")
Observable<String> upload(@PartMap Map<String, RequestBody> params);
```

**参数的规范**
>	// 使用 PartMap 
> partMap.put("参数名\"; filename=\"文件名\"", file);

***filename要写，发现不写会上传失败***，至于文件名无所谓，服务器需要知道文件名那就另当别论

这里有个规范说得比较清楚 [春上冰月](http://www.caoyue.com.cn/blog/2016/02/12/How-to-upload-file-with-retrofit2/)
```java
RequestBody body = RequestBody.create(MediaType.parse("image/*"), file);
Map<String, RequestBody> partMap = new HashMap<>();
partMap.put("thumb\"; filename=\"img.jpg\"", body);
```
**上传拦截监听进度的接口**
```java
public interface UploadListener {

    /**
     * @param bytesWritten  上传的字节
     * @param contentLength 总字节
     */
    void onRequestProgress(long bytesWritten, long contentLength);

}
```
**上传拦截器**
```java
public class UploadInterceptor implements Interceptor {

    private UploadListener progressListener;

    public UploadInterceptor(UploadListener progressListener) {
        this.progressListener = progressListener;
    }

    @Override
    public Response intercept(Interceptor.Chain chain) throws IOException {
        Request originalRequest = chain.request();

        if (originalRequest.body() == null) {
            return chain.proceed(originalRequest);
        }

        Request progressRequest = originalRequest.newBuilder()
                .method(originalRequest.method(), new UploadRequestBody(originalRequest.body(), progressListener))
                .build();

        return chain.proceed(progressRequest);
    }
}
```
**继承RequestBody**
文件上传过程的进度中就从RequestBody获取
```java
public class UploadRequestBody extends RequestBody {

    protected RequestBody delegate;
    protected UploadListener listener;
    protected CountingSink countingSink;

    public UploadRequestBody(RequestBody delegate, UploadListener listener) {
        this.delegate = delegate;
        this.listener = listener;
    }

    @Override
    public MediaType contentType() {
        return delegate.contentType();
    }

    @Override
    public long contentLength() {
        try {
            return delegate.contentLength();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return -1;
    }

    @Override
    public void writeTo(BufferedSink sink) throws IOException {
        countingSink = new CountingSink(sink);
        BufferedSink bufferedSink = Okio.buffer(countingSink);
        delegate.writeTo(bufferedSink);
        bufferedSink.flush();
    }

    protected final class CountingSink extends ForwardingSink {

        private long bytesWritten = 0;

        public CountingSink(Sink delegate) {
            super(delegate);
        }

        @Override
        public void write(Buffer source, long byteCount) throws IOException {
            super.write(source, byteCount);
            bytesWritten += byteCount;
            listener.onRequestProgress(bytesWritten, contentLength());
        }
    }
}
```
**拦截器的配置就如同下载的一样**
```java
OkHttpClient okHttpClient = new OkHttpClient()
        .newBuilder()
        .addInterceptor(new UploadInterceptor(listener))
        .build();
```

------
**ps:**继承RequestBody的来源一个大神 **okhttp-utils** (https://github.com/hongyangAndroid/okhttp-utils)，这个封装好的okhttp，也是用过还不错。

