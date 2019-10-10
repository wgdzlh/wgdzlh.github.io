---
title: 使用自签名证书给安卓APP提供HTTPS后端服务
toc: true
date: 2019-10-09 19:57:52
updated: 2019-10-09 19:57:52
categories: 安卓开发
tags: [Java, Android]
thumbnail: /gallery/https_vs_http.jpg

---

在安卓开发中，我们希望可以用更安全的方式连接后端服务，且近几年较新的安卓Framework也一直在引导开发者使用https安全链接。本文介绍了一种使用自签名证书的安卓https链接方式。
<!-- more -->

## 需求背景

关于Http与Https的区别，大家应该很熟悉，就不再赘述了。苹果iOS在好几年前就已强推Https，而安卓这边紧随其后，默认开发配置下也要求App只能使用Https，虽说目前最新版（9.0）仍然可以在manifest里设置一下规避掉，也只是过渡期的权宜之计罢了。

Https其实大家都想上，但证书问题却是个老大难。一方面，后端服务器可能没有域名，只有个IP，无法自由申请免费证书（可以[给IP申请证书](https://segmentfault.com/a/1190000018370382)，但有很多限制，且可能收费）。另一方面，自签名证书很方便，但却没经过授权机构根证书签名，默认情况下得不到安卓系统的信任。因而，市面上仍有大量App使用不安全的Http协议连接Web后端服务。

## 解决方法
通过多方搜索查找，综合[官方文档](https://developer.android.com/training/articles/security-ssl#UnknownCa)，及[相关博客](https://jebware.com/blog/?p=340)，可以使用以下两步，验证自签名CA证书。另外，需注意生成证书时，应加入Subject Alternative Name参数，域名或者IP都行，具体参考这个[SO回答](https://stackoverflow.com/a/8744717/8578416)。

PS：代码中的cer证书放入项目asset文件夹，获取方式可以参考[相关博客](https://jebware.com/blog/?p=340)，但不需要转为pem格式。

### 自定义证书验证器TrustManager


``` java
package your.project.io;

import java.io.BufferedInputStream;
import java.security.KeyStore;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;
import java.util.Arrays;

import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManager;
import javax.net.ssl.TrustManagerFactory;
import javax.net.ssl.X509TrustManager;

import your.project.Application;


class MyTrustManager {

    private static SSLSocketFactory mSSLSocketFactory;
    private static X509TrustManager mTrustManager;

    static {
        try {
            KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
            keyStore.load(null, null);
            try (BufferedInputStream bis = new BufferedInputStream(
                    Application.getApplicationContext().getAssets().open("ca/server.cer"))) {
                Certificate ca = CertificateFactory.getInstance("X.509").generateCertificate(bis);
                keyStore.setCertificateEntry("ca", ca);
            }
            TrustManagerFactory trustManagerFactory = TrustManagerFactory
                    .getInstance(TrustManagerFactory.getDefaultAlgorithm());
            trustManagerFactory.init(keyStore);
            TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
            if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
                throw new IllegalStateException("Unexpected default trust managers:"
                        + Arrays.toString(trustManagers));
            }
            mTrustManager = (X509TrustManager) trustManagers[0];
            SSLContext sslContext = SSLContext.getInstance("TLS");
            sslContext.init(null, trustManagers, null);
            mSSLSocketFactory = sslContext.getSocketFactory();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static SSLSocketFactory getSocketFactory() {
        return mSSLSocketFactory;
    }

    static X509TrustManager getTrustManager() {
        return mTrustManager;
    }
}
```

### 在连接代理中使用定制的TrustManager

在Http连接代理中设定`TrustManager`，这里以okhttp3第三方库的HttpClient为例：

``` java
static final OkHttpClient client = new OkHttpClient.Builder()
            .sslSocketFactory(MyTrustManager.getSocketFactory(), MyTrustManager.getTrustManager())
            // not safe, should specify subject alt name in server's ca.
            //.hostnameVerifier((hostname, session) -> Constants.Net.HOST.equals(hostname))
            .connectTimeout(10, TimeUnit.SECONDS)
            .writeTimeout(20, TimeUnit.SECONDS)
            .readTimeout(20, TimeUnit.SECONDS).build();

```

不出意外的话，App端应该可以正常用Https访问你的Web服务了。

