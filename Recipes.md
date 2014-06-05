We've written some recipes that demonstrate how to solve common problems with OkHttp. Read through them to learn about how everything works together. Cut-and-paste these examples freely; that's what they're for.

#### [Synchronous Get](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/SynchronousGet.java)

Download a file, print its headers, and print its response body as a string.

The `string()` method on response body is convenient and efficient for small documents. But if the response body is large (greater than 1 MiB), avoid `string()` because it will load the entire document into memory. In that case, prefer to process the body as a stream.

```java
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    Headers responseHeaders = response.headers();
    for (int i = 0; i < responseHeaders.size(); i++) {
      System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
    }

    System.out.println(response.body().string());
  }
```

#### [Asynchronous Get](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/AsynchronousGet.java)

Download a file on a worker thread, and get called back when the response is readable. The callback is made after the response headers are ready. Reading the response body may still block. OkHttp doesn't currently offer asynchronous APIs to receive a response body in parts.

```java
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    client.newCall(request).enqueue(new Callback() {
      @Override public void onFailure(Request request, Throwable throwable) {
        throwable.printStackTrace();
      }

      @Override public void onResponse(Response response) throws IOException {
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

        Headers responseHeaders = response.headers();
        for (int i = 0; i < responseHeaders.size(); i++) {
          System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
        }

        System.out.println(response.body().string());
      }
    });
  }
```

#### [Posting a String](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/PostString.java)

Use an HTTP POST to send a request body to a service. This example posts a markdown document to a web service that renders markdown as HTML. Because the entire request body is in memory simultaneously, avoid posting large (greater than 1 MiB) documents using this API.

```java
  public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    String postBody = ""
        + "Releases\n"
        + "--------\n"
        + "\n"
        + " * _1.0_ May 6, 2013\n"
        + " * _1.1_ June 15, 2013\n"
        + " * _1.2_ August 11, 2013\n";

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, postBody))
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```

#### [Post Streaming](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/PostStreaming.java)

Here we `POST` a request body as a stream. The content of this request body is being generated as it's being written. This example streams directly into the [Okio](https://github.com/square/okio) buffered sink. Your programs may prefer an `OutputStream`, which you can get from `BufferedSink.outputStream()`.

```java
  public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    RequestBody requestBody = new RequestBody() {
      @Override public MediaType contentType() {
        return MEDIA_TYPE_MARKDOWN;
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        sink.writeUtf8("Numbers\n");
        sink.writeUtf8("-------\n");
        for (int i = 2; i <= 997; i++) {
          sink.writeUtf8(String.format(" * %s = %s\n", i, factor(i)));
        }
      }

      private String factor(int n) {
        for (int i = 2; i < n; i++) {
          int x = n / i;
          if (x * i == n) return factor(x) + " × " + i;
        }
        return Integer.toString(n);
      }
    };

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(requestBody)
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```

#### [Posting a File](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/PostFile.java)

It's easy to use a file as a request body.

```java
  public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    File file = new File("README.md");

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, file))
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```

#### [Parse a JSON Response With Gson](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/ParseResponseWithGson.java)

[Gson](http://code.google.com/p/google-gson/) is a handy API for converting between JSON and Java objects. Here we're using it to decode a JSON response from a GitHub API.

Note that `ResponseBody.charStream()` uses the `Content-Type` response header to select which charset to use when decoding the response body. It defaults to `UTF-8` if no charset is specified.

```java
  private final OkHttpClient client = new OkHttpClient();
  private final Gson gson = new Gson();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://api.github.com/gists/c2a7c39532239ff261be")
        .build();
    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    Gist gist = gson.fromJson(response.body().charStream(), Gist.class);
    for (Map.Entry<String, GistFile> entry : gist.files.entrySet()) {
      System.out.println(entry.getKey());
      System.out.println(entry.getValue().content);
    }
  }

  static class Gist {
    Map<String, GistFile> files;
  }

  static class GistFile {
    String content;
  }
```

#### [Response Caching](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/CacheResponse.java)

To cache responses, you'll need a cache directory that you can read and write to, and a limit on the cache's size. The cache directory should be private, and untrusted applications should not be able to read its contents!

It is an error to have multiple caches accessing the same cache directory simultaneously. Most applications should call `new OkHttp()` exactly once, configure it with their cache, and use that same instance everywhere. Otherwise the two cache instances will stomp on each other, corrupt the response cache, and possibly crash your program.

Response caching uses HTTP headers for all configuration. You can add request headers like `Cache-Control: max-stale=3600` and OkHttp's cache will honor them. Your webserver configures how long responses are cached with its own response headers, like `Cache-Control: max-age=9600`. There are cache headers to force a cached response, force a network response, or force the network response to be validated with a conditional GET.

```java
  private final OkHttpClient client;

  public CacheResponse(File cacheDirectory) throws Exception {
    int cacheSize = 10 * 1024 * 1024; // 10 MiB
    Cache cache = new Cache(cacheDirectory, cacheSize);

    client = new OkHttpClient();
    client.setCache(cache);
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    Response response1 = client.newCall(request).execute();
    if (!response1.isSuccessful()) throw new IOException("Unexpected code " + response1);

    String response1Body = response1.body().string();
    System.out.println("Response 1 response:          " + response1);
    System.out.println("Response 1 cache response:    " + response1.cacheResponse());
    System.out.println("Response 1 network response:  " + response1.networkResponse());

    Response response2 = client.newCall(request).execute();
    if (!response2.isSuccessful()) throw new IOException("Unexpected code " + response2);

    String response2Body = response2.body().string();
    System.out.println("Response 2 response:          " + response2);
    System.out.println("Response 2 cache response:    " + response2.cacheResponse());
    System.out.println("Response 2 network response:  " + response2.networkResponse());

    System.out.println("Response 2 equals Response 1? " + response1Body.equals(response2Body));
  }
```

#### [Canceling a Call](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/CancelCall.java)

Use `Call.cancel()` to stop an ongoing call immediately. If a thread is currently writing a request or reading a response, it will receive an `IOException`. Use this to conserve the network when a call is no longer necessary; for example when your user navigates away from an application. Both synchronous and asynchronous calls can be canceled.

You can cancel multiple requests simultaneously with tags. Assign a tag when you build a request with `RequestBuilder.tag(tag)`. You can then use `OkHttpClient.cancel(tag)` to cancel all calls with that tag.

```java
  private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/2") // This URL is served with a 2 second delay.
        .build();

    final long startNanos = System.nanoTime();
    final Call call = client.newCall(request);

    // Schedule a job to cancel the call in 1 second.
    executor.schedule(new Runnable() {
      @Override public void run() {
        System.out.printf("%.2f Canceling call.%n", (System.nanoTime() - startNanos) / 1e9f);
        call.cancel();
        System.out.printf("%.2f Canceled call.%n", (System.nanoTime() - startNanos) / 1e9f);
      }
    }, 1, TimeUnit.SECONDS);

    try {
      System.out.printf("%.2f Executing call.%n", (System.nanoTime() - startNanos) / 1e9f);
      Response response = call.execute();
      System.out.printf("%.2f Call was expected to fail, but completed: %s%n",
          (System.nanoTime() - startNanos) / 1e9f, response);
    } catch (IOException e) {
      System.out.printf("%.2f Call failed as expected: %s%n",
          (System.nanoTime() - startNanos) / 1e9f, e);
    }
  }
```

#### [Timeouts](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/ConfigureTimeouts.java)

Use timeouts to fail a call when its peer is unreachable. Network partitions can be due to client connectivity problems, server availability problems, or anything between. OkHttp supports connect, read, and write timeouts.

```java
  private final OkHttpClient client;

  public ConfigureTimeouts() throws Exception {
    client = new OkHttpClient();
    client.setConnectTimeout(10, TimeUnit.SECONDS);
    client.setWriteTimeout(10, TimeUnit.SECONDS);
    client.setReadTimeout(30, TimeUnit.SECONDS);
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/2") // This URL is served with a 2 second delay.
        .build();

    Response response = client.newCall(request).execute();
    System.out.println("Response completed: " + response);
  }
```

#### [Per-call Configuration](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/PerCallSettings.java)

All the HTTP client configuration lives in `OkHttpClient` including proxy settings, timeouts, and caches. When you need to change the configuration of a single call, clone the `OkHttpClient`. This returns a shallow copy that you can customize independently. In the example below, we make one request with a 500 ms timeout and another with a 3000 ms timeout.

```java
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/1") // This URL is served with a 1 second delay.
        .build();

    try {
      Response response = client.clone() // Clone to make a customized OkHttp for this request.
          .setReadTimeout(500, TimeUnit.MILLISECONDS)
          .newCall(request)
          .execute();
      System.out.println("Response 1 succeeded: " + response);
    } catch (IOException e) {
      System.out.println("Response 1 failed: " + e);
    }

    try {
      Response response = client.clone() // Clone to make a customized OkHttp for this request.
          .setReadTimeout(3000, TimeUnit.MILLISECONDS)
          .newCall(request)
          .execute();
      System.out.println("Response 2 succeeded: " + response);
    } catch (IOException e) {
      System.out.println("Response 2 failed: " + e);
    }
  }
```