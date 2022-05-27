---
layout: post
title: "如何判断SSL证书的私钥和证书匹配"
date: 2019-05-22
excerpt: "防止有人篡改证书或者私钥，导致网关无法解析，所以需要验证私钥和证书是否匹配。开始以为Java会有这种方法直接校验，网上找了一圈都没找到。翻了 `X509Certificate` 的源码，发现可以获取到公钥，那么也可以通过私钥获取到公钥，所以判断两个公钥是否相等就可以了。"
categories: 
- Java
tags: 
- SSL
comments: true
---

### 前言
防止有人篡改证书或者私钥，导致网关无法解析，所以需要验证私钥和证书是否匹配。开始以为Java会有这种方法直接校验，网上找了一圈都没找到。翻了 `X509Certificate` 的源码，发现可以获取到公钥，那么也可以通过私钥获取到公钥，所以判断两个公钥是否相等就可以了。

### 通过证书获取公钥
> 网上资料很多，一搜就有了

```java
public static String getPublicValByCert(String cert) {

    try {
        CertificateFactory certificateFactory = CertificateFactory
                .getInstance("X.509");
        X509Certificate x509certificate = (X509Certificate) certificateFactory
                .generateCertificate(new ByteArrayInputStream(cert.getBytes()));
        return getValue(x509certificate.getPublicKey());
    } catch (Exception e) {
        log.error("解析公钥异常", e);
    }

    return null;
}

private static String getValue(PublicKey publicKey) {
    BASE64Encoder bse = new BASE64Encoder();
    return bse.encode(publicKey.getEncoded());
}
```    


### 通过私钥获取公钥

按照网上的教程，大部分都是将PKCS#1转换成PKCS#8，然后通过 `PKCS8EncodedKeySpec` 获取到公钥。但是根本不管用。
> 参考的网址：[https://stackoverflow.com/questions/8434428/get-public-key-from-private-in-java](https://stackoverflow.com/questions/8434428/get-public-key-from-private-in-java) 

按照上面的方式总是报错，大概就是第一行不是合法的格式。如果不去掉第一行和最后一行，也还是不能解析。

尝试了几百次了吧，后来看见一篇获取 `PEM` 的文章，才柳暗花明。  
贴上代码，如果以后遇到同样的坑，又不用乱找了。

```java
public static String getPublicValByKey(String key) {
    Security.addProvider(new BouncyCastleProvider());
    try {
        StringReader stringReader = new StringReader(key);
        PEMParser pemParser = new PEMParser(stringReader);
        JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
        Object object = pemParser.readObject();
        KeyPair kp = converter.getKeyPair((PEMKeyPair) object);
        return getValue(kp.getPublic());
    } catch (Exception e) {
        log.error("解析私钥异常", e);
    }

    return null;
}
```


### 总结

##### PKCS1

PKCS#1结构仅为RSA设计

```
-----BEGIN RSA PUBLIC KEY-----
BASE64 ENCODED DATA
-----END RSA PUBLIC KEY-----
```

##### PKCS8

X509,SSL支持的算法不仅仅是RSA，因此产生了更具有通用性的PKCS#8

```
-----BEGIN PUBLIC KEY-----
BASE64 ENCODED DATA
-----END PUBLIC KEY-----
```

与PKCS#1相比将文件包含的加密算法和Key分开存储，因此可以存储其他加密算法的Key




