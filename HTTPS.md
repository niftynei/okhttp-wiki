OkHttp attempts to balance two competing concerns:

 * **Connectivity** to as many hosts as possible. That includes advanced hosts that run the latest versions of [boringssl](https://boringssl.googlesource.com/boringssl/), less up-to-date hosts running older versions of [OpenSSL](https://www.openssl.org/).
 * **Security** of the connection, including verification of the remote webserver and the privacy of its content.

Specific security vs. connectivity decisions are implemented by [ConnectionSpec](http://square.github.io/okhttp/javadoc/com/squareup/okhttp/ConnectionSpec.html). When negotiating a connection to an HTTPS server, OkHttp needs to know which [TLS versions](http://square.github.io/okhttp/javadoc/com/squareup/okhttp/TlsVersion.html) and [cipher suites](http://square.github.io/okhttp/javadoc/com/squareup/okhttp/CipherSuite.html) to offer. A client that wants to maximize connectivity would include obsolete TLS versions and weak-by-design cipher suites. A strict client that wants to maximize security would be limited to only the latest TLS version and strongest cipher suites.

OkHttp includes three built-in connection specs:

 * `MODERN_TLS` is a secure configuration that connects to modern HTTPS servers.
 * `COMPATIBLE_TLS` is a secure configuration that connects to secure–but–not–current HTTPS servers.
 * `CLEARTEXT` is an insecure configuration that is used for `http://` URLs.

By default, for `https://` URLs, OkHttp will attempt a `MODERN_TLS` connection, and fall back to `COMPATIBLE_TLS` connection if the modern configuration fails.

The TLS versions and cipher suites in each spec can change with each release. For example, in OkHttp 2.2 we dropped support for [SSL 3.0](http://googleonlinesecurity.blogspot.ca/2014/10/this-poodle-bites-exploiting-ssl-30.html) in response to the POODLE attack. And in OkHttp 2.3 we dropped support for [RC4](http://en.wikipedia.org/wiki/RC4#Security). As with your desktop web browser, staying up-to-date with OkHttp is the best way to stay secure.

You can build your own connection spec with a custom set of TLS versions and cipher suites. For example, this configuration is limited to three highly-regarded cipher suites. Its drawback is that it requires Android 5.0+ or and a similarly up-to-date webserver.

```java
ConnectionSpec spec = new ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)  
    .tlsVersions(TlsVersion.TLS_1_2)
    .cipherSuites(
          CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
          CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
          CipherSuite.TLS_DHE_RSA_WITH_AES_128_GCM_SHA256)
    .build();

OkHttpClient client = ...
client.setConnectionSpecs(Collections.singletonList(spec));
```

### [Certificate Pinning](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/CertificatePinning.java)

By default, OkHttp trusts the certificate authorities of the host platform. This strategy maximizes connectivity, but it is subject to certificate authority attacks such as the [2011 attack](http://www.computerworld.com/article/2510951/cybercrime-hacking/hackers-spied-on-300-000-iranians-using-fake-google-certificate.html) on DigiNotar. It also assumes your HTTPS servers’ certificates are signed by a certificate authority.

Use [CertificatePinner](http://square.github.io/okhttp/javadoc/com/squareup/okhttp/CertificatePinner.html) to constrain which certificate authorities are trusted. Certificate pinning increases security, but limits your server team's abilities to update their TLS certificates. **Do not use certificate pinning without the blessing of your server's TLS administrator!**

```
  public CertificatePinning() {
    client = new OkHttpClient();
    client.setCertificatePinner(
        new CertificatePinner.Builder()
            .add("publicobject.com", "sha1/DmxUShsZuNiqPQsX2Oi9uv2sCnw=")
            .add("publicobject.com", "sha1/SXxoaOSEzPC6BgGmxAt/EAcsajw=")
            .add("publicobject.com", "sha1/blhOM3W9V/bVQhsWAcLYwPU6n24=")
            .add("publicobject.com", "sha1/T5x9IXmcrQ7YuQxXnxoCmeeQ84c=")
            .build());
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://publicobject.com/robots.txt")
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    for (Certificate certificate : response.handshake().peerCertificates()) {
      System.out.println(CertificatePinner.pin(certificate));
    }
  }
```

### Customizing Trusted Certificates

The [CustomTrust](https://github.com/square/okhttp/blob/master/samples/guide/src/main/java/com/squareup/okhttp/recipes/CustomTrust.java) sample shows how to replace the host platform’s certificate authorities with your own set. As above, **do not use custom certificates without the blessing of your server's TLS administrator!**

```
  private final OkHttpClient client;

  public CustomTrust() {
    client = new OkHttpClient();
    SSLContext sslContext = sslContextForTrustedCertificates(trustedCertificatesInputStream());
    client.setSslSocketFactory(sslContext.getSocketFactory());
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://publicobject.com/helloworld.txt")
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    Headers responseHeaders = response.headers();
    for (int i = 0; i < responseHeaders.size(); i++) {
      System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
    }

    System.out.println(response.body().string());
  }

  private InputStream trustedCertificatesInputStream() {
    ...
  }

  public SSLContext sslContextForTrustedCertificates(InputStream in) {
    ...
  }
```