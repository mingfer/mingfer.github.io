---
title: "Java 下常用密码算法的使用"
subtitle: "如何使用 BouncyCastle 算法库"
date: 2019-09-29 13:00:00
author: "mingfer"
header-img: "img/post/bg-2.jpg"
catalog: true
tags: 
    - "密码学"
    - "BouncyCastle"
---

本文介绍了 BouncyCastle 库中的 JCE 接口的具体用法。阅读本文之前，我们假设您已经对常用的加密算法，如 RSA，DES/3DES，SM2，SM3，SM4 已经有一定的了解。如果您并不了解这些算法的具体功能，可以先阅读对应的算法规范，这些算法规范可以在 [IETF](https://tools.ietf.org) 找到。 

[BouncyCastle](http://www.bouncycastle.org/) 是一个有着 20 余年历史的开源算法库，它提供了轻量级的支持 Java 的算法接口，实现了 Java 中的 JCE 接口和 JCA 接口。下面为您介绍的是如何使用 BouncyCastle 提供的 JCE 接口，BouncyCastle 支持的算法信息可以在 [这里](http://www.bouncycastle.org/specifications.html) 获取。

### 环境配置

#### 解决出口限制

Java 的 JCE 接口在设计上采用了服务提供者的设计模式， Java 定义了标准的 JCE 接口，由第三方的密码服务提供商实现这些接口，在 JDK 中默认的 JCE 提供者是 SUN 实现的。由于 Oracle 是注册于美国的公司，必须遵守美国的出口限制规范，所以在使用 JCE 时可能会出现高强度的密钥（如 AES-256 密钥）报密钥长度非法错误的情况。

为了解决这个问题，我们需要将运行代码的 jdk/jre 配置成无限制的。对于 JDK6/JDK7/JDK1.8.0_151 范围内的 Java 版本我们需要在 Oracle 官方下载无限制策略文件，然后替换掉 `${java_home}/jre/lib/security/` 下面的 `local_policy.jar` 和 `US_export_policy.jar`。各版本的下载地址：

- [jce_policy-6.zip](https://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html)
- [UnlimitedJCEPolicyJDK7.zip](https://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html)
- [jce_policy-8.zip](https://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)

对于高于 JDK1.8.0_151 的 JDK 版本，只需要修改  `${java_home}/jre/lib/security/`  中 `java.security` 的 `crypto.policy` 配置为 `unlimited` 即可。

#### 注册 Provider

在使用 BouncyCastle 的库之前，需要注册 BouncyCastle 的 Provider。由于每次创建 Provider 对象的时候都会对 Provider 中的所有 `.class` 进行验签，请务必注意不要频繁的创建 Provider 对象，否则会导致代码的执行性能极低。一个比较好的办法是，在一个类 （如 `ProviderUtils`）中声明和注册 Provider，然后所有使用到 Provider 的地方都使用这个类中创建的 Provider 对象。示例如下：

```java
import org.bouncycastle.jce.provider.BouncyCastleProvider;

import java.security.Provider;
import java.security.Security;

public class ProviderUtils {

    public static final Provider PROVIDER = new BouncyCastleProvider();

    static {
        Security.addProvider(PROVIDER);
    }
}

```

### 密钥管理

密钥是整个密码服务体系中的关键数据，JCE 中有三种方式去管理不同状态的密钥。

#### 密钥生成

当您需要创建一把新的密钥的时候，您可以使用密钥生成服务去创建对称密钥或非对称密钥对。

**创建非对称密钥**

```java
import org.bouncycastle.jce.ECNamedCurveTable;
import org.bouncycastle.jce.spec.ECParameterSpec;
import org.junit.Assert;
import org.junit.Test;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.PrivateKey;
import java.security.PublicKey;

public class KeyPairUtils {

    @Test
    public void generateRSAKeyPair() throws Exception {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA", ProviderUtils.PROVIDER);

        // 仅指定密钥长度
        generator.initialize(2048);

        // 指定密钥长度和公钥指数
//        RSAKeyGenParameterSpec spec = new RSAKeyGenParameterSpec(2048, new BigInteger("65537"));
//        generator.initialize(spec);
        KeyPair keyPair = generator.generateKeyPair();
        PublicKey publicKey = keyPair.getPublic();
        PrivateKey privateKey = keyPair.getPrivate();
        Assert.assertEquals("RSA", publicKey.getAlgorithm());
        Assert.assertEquals("RSA", privateKey.getAlgorithm());
    }

    @Test
    public void generateECKeyPair() throws Exception {
        // 生成 SM2 密钥对的曲线
        ECParameterSpec sm2p256v1 = ECNamedCurveTable.getParameterSpec("sm2p256v1");

        // 比较常用的 EC 算法曲线
        ECParameterSpec prime256v1 = ECNamedCurveTable.getParameterSpec("prime256v1");

        KeyPairGenerator generator = KeyPairGenerator.getInstance("EC", ProviderUtils.PROVIDER);
        generator.initialize(prime256v1);
        KeyPair keyPair = generator.generateKeyPair();
        PublicKey publicKey = keyPair.getPublic();
        PrivateKey privateKey = keyPair.getPrivate();
        Assert.assertEquals("EC", publicKey.getAlgorithm());
        Assert.assertEquals("EC", privateKey.getAlgorithm());
    }
    
    @Test
    public void getAllECCurveName() {
        Enumeration names = ECNamedCurveTable.getNames();
        while (names.hasMoreElements()) {
            System.out.println(names.nextElement());
        }
    }
}

```

**创建对称密钥**

```java
import org.junit.Assert;
import org.junit.Test;

import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;

public class SecretKeyTest {

    @Test
    public void generateSecretKey() throws Exception {
        // 指定密钥的算法为 SM4
        KeyGenerator generator = KeyGenerator.getInstance("SM4", ProviderUtils.PROVIDER);
        // 指定生成的密钥长度为 128 bit
        generator.init(128); 
        SecretKey secretKey = generator.generateKey();
        Assert.assertEquals("SM4", secretKey.getAlgorithm());
    }

}

```

BouncyCastle 支持的对称密钥及长度为：

| 算法   | 长度 |
| ------ | ---- |
| AES    | 128  |
| AES    | 192  |
| AES    | 256  |
| DES    | 64   |
| DESede | 128  |
| DESede | 192  |
| SM4    | 128  |

#### 密钥工厂

密钥工厂用于将字符串形式的密钥，文件中的密钥或其他形式的密钥转换为密钥对象。

**非对称密钥**

下面的代码中需要注意，EC（SM2）的公钥的第一个字节用于表示公钥的编码方式，一般来说是 `0x04` 无编码。如果用于转换的密钥值没有编码标识，可以手动加上 `04`。

```java
import cn.pdgp.plsql.util.Hex;
import org.bouncycastle.jce.ECNamedCurveTable;
import org.bouncycastle.jce.spec.ECParameterSpec;
import org.bouncycastle.jce.spec.ECPrivateKeySpec;
import org.bouncycastle.jce.spec.ECPublicKeySpec;
import org.junit.Assert;
import org.junit.Test;

import java.math.BigInteger;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.spec.RSAPrivateKeySpec;
import java.security.spec.RSAPublicKeySpec;

public class KeyPairFactoryTest {

    @Test
    public void createRSAKeyPair() throws Exception {
        KeyFactory factory = KeyFactory.getInstance("RSA", ProviderUtils.PROVIDER);
        BigInteger modulus = new BigInteger("154471992999058139479994460025815654498183391593444870454838266974581244599191659985455957889064163942388409487313472074598227824609910604156744751985833898809065078785899074110993629452358669379496163284362583792866500058660069050752020922895749548342185553141417346777273482310707415185758164008066298773949",10);
        BigInteger publicExponent = new BigInteger("65537");
        RSAPublicKeySpec rsaPublicKeySpec = new RSAPublicKeySpec(modulus,publicExponent);
        PublicKey publicKey = factory.generatePublic(rsaPublicKeySpec);
        BigInteger privateExponent = new BigInteger("24953766420205815381764520016071994967304996670579990593182061010725111564027070269710579156377653900210050677360692873548856950717077735724971492275722465522175892883197573916804276397143284954594245180776141869860033925480138858143033802945465036705957639063440190950861284456594945244826689811470380537909",10);
        RSAPrivateKeySpec rsaPrivateKeySpec = new RSAPrivateKeySpec(modulus,privateExponent);
        PrivateKey privateKey = factory.generatePrivate(rsaPrivateKeySpec);
        Assert.assertEquals("RSA", publicKey.getAlgorithm());
        Assert.assertEquals("RSA", privateKey.getAlgorithm());
    }

    @Test
    public void createECKeyPair() throws Exception {
        final byte[] publicKeyValue = Hex.decode("04b75d23311d899784e6f706571841b8d63b933b56f114a3e3a66449277e60f0c339ab6594b72f346ca8cbdbf3d3c0589d1042ccd3d5e6388d590000fe5e546d5c");
        final byte[] privateKeyValue = Hex.decode("2AB4A556B0D396969B5F7479893FE1651ECEE7192F3A33FD9302DC30A85A6F03");
        ECParameterSpec prime256v1 = ECNamedCurveTable.getParameterSpec("prime256v1");
        ECPublicKeySpec publicKeySpec = new ECPublicKeySpec(prime256v1.getCurve().decodePoint(publicKeyValue), prime256v1);
        ECPrivateKeySpec privateKeySpec = new ECPrivateKeySpec(new BigInteger(1, privateKeyValue), prime256v1);
        KeyFactory factory = KeyFactory.getInstance("EC", ProviderUtils.PROVIDER);
        PublicKey publicKey = factory.generatePublic(publicKeySpec);
        PrivateKey privateKey = factory.generatePrivate(privateKeySpec);
        Assert.assertEquals("EC", publicKey.getAlgorithm());
        Assert.assertEquals("EC", privateKey.getAlgorithm());
    }
}
```

**对称密钥**

事实上，您不需要通过工厂去转换对称密钥（除非您需要对密钥做特殊的处理），直接使用构建出的 `SecretKeySpec` 对象即可，`SecretKeySpec` 实现了 `SecretKey` 的所有接口。**注意**：这里不会判断密钥的长度等信息是否合法，而是在使用的时候由使用密钥的接口进行判断。

```java
import org.junit.Assert;
import org.junit.Test;

import javax.crypto.SecretKey;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.SecretKeySpec;

public class SecretKeyFactoryTest {


    @Test
    public void createSecretKey() throws Exception {
        SecretKeySpec spec = new SecretKeySpec("1234567812354678".getBytes(), "AES");
        SecretKeyFactory factory = SecretKeyFactory.getInstance("AES", ProviderUtils.PROVIDER);
        SecretKey secretKey = factory.generateSecret(spec);
        Assert.assertEquals("AES", secretKey.getAlgorithm());
    }
}

```

#### 密钥存储



### 加解密 Cipher

JCE 中通过 Cipher 接口实现对称算法和非对称算法的加解密操作。

支持的对称算法，算法模式和填充方式主要有（表中三列数据可任意组合）：

| 算法   | 模式 | 填充方式                                      |
| ------ | ---- | --------------------------------------------- |
| AES    | ECB  | NoPadding                                     |
| DES    | CBC  | PKCS5Padding/PKCS7Padding                     |
| DESede | GCM  | ISO10126Padding/ISO7816-4Padding              |
| SM4    | CTR  | ISO7816-4Padding/ISO9797-1Padding （即 PBOC） |
|        |      | ZeroBytePadding                               |
|        |      | X9.23Padding/X923Padding                      |

```java
import org.junit.Assert;
import org.junit.Test;

import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.PrivateKey;
import java.security.PublicKey;

public class CipherTest {

    @Test
    public void cipherWithDESede() throws Exception {
        final byte[] data = "this is a test data.".getBytes();
        SecretKeySpec secretKey = new SecretKeySpec("1234567812345678".getBytes(), "DESede");
        Cipher cipher = Cipher.getInstance("DESede/CBC/ISO7816-4Padding", ProviderUtils.PROVIDER);
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, new IvParameterSpec("12345678".getBytes()));
        byte[] ciphertext = cipher.doFinal(data);

        cipher.init(Cipher.DECRYPT_MODE, secretKey, new IvParameterSpec("12345678".getBytes()));
        byte[] plaintext = cipher.doFinal(ciphertext);

        Assert.assertArrayEquals(data, plaintext);
    }

    @Test
    public void cipherWithRSA() throws Exception {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA", ProviderUtils.PROVIDER);
        generator.initialize(2048);
        KeyPair keyPair = generator.generateKeyPair();
        PublicKey publicKey = keyPair.getPublic();
        PrivateKey privateKey = keyPair.getPrivate();

        final byte[] data = "this is a test data.".getBytes();
        Cipher cipher = Cipher.getInstance("RSA/NONE/PKCS1Padding", ProviderUtils.PROVIDER);
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        final byte[] ciphertext = cipher.doFinal(data);

        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        byte[] plaintext = cipher.doFinal(ciphertext);
        Assert.assertArrayEquals(data, plaintext);
    }
}

```



### 签名验签 Signature

BouncyCastle 几乎支持了所有的签名验签算法，常用的包括：

- MD5withRSA
- SHA1withRSA
- SHA224withRSA
- SHA256withRSA
- SHA384withRSA
- SHA512withRSA
- SHA512(224)withRSA
- SHA512(256)withRSA
- SHA3-224withRSA
- SHA3-256withRSA
- SHA3-384withRSA
- SHA3-512withRSA
- SHA1withRSAandMGF1
- SHA256withRSAandMGF1
- SHA384withRSAandMGF1
- SHA512withRSAandMGF1
- SHA512(224)withRSAandMGF1
- SHA512(256)withRSAandMGF1
- SHA256withSM2
- SM3withSM2

```java
import org.bouncycastle.jcajce.spec.SM2ParameterSpec;
import org.bouncycastle.jce.ECNamedCurveTable;
import org.bouncycastle.jce.spec.ECParameterSpec;
import org.junit.Assert;
import org.junit.Test;

import java.security.*;

public class SignatureTest {

    @Test
    public void signatureWithRSA() throws Exception {

        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA", ProviderUtils.PROVIDER);
        generator.initialize(2048);
        KeyPair keyPair = generator.generateKeyPair();
        PublicKey publicKey = keyPair.getPublic();
        PrivateKey privateKey = keyPair.getPrivate();

        final byte[] data = "this is a test data.".getBytes();
        Signature signature = Signature.getInstance("SHA1withRSA", ProviderUtils.PROVIDER);
        signature.initSign(privateKey);
        signature.update(data);
        byte[] sign = signature.sign();

        signature.initVerify(publicKey);
        signature.update(data);
        boolean result = signature.verify(sign);

        Assert.assertTrue(result);
    }


    @Test
    public void signatureWithSM2() throws Exception {
        ECParameterSpec sm2p256v1 = ECNamedCurveTable.getParameterSpec("sm2p256v1");
        KeyPairGenerator generator = KeyPairGenerator.getInstance("EC", ProviderUtils.PROVIDER);
        generator.initialize(sm2p256v1);
        KeyPair keyPair = generator.generateKeyPair();
        PublicKey publicKey = keyPair.getPublic();
        PrivateKey privateKey = keyPair.getPrivate();

        final byte[] data = "this is a test data.".getBytes();
        Signature signature = Signature.getInstance("SM3WithSM2", ProviderUtils.PROVIDER);
        // UserID
        SM2ParameterSpec spec = new SM2ParameterSpec("1234567812345678".getBytes());
        signature.initSign(privateKey);
        signature.setParameter(spec);
        signature.update(data);
        byte[] sign = signature.sign();

        signature.initVerify(publicKey);
        signature.update(data);
        boolean result = signature.verify(sign);

        Assert.assertTrue(result);
    }
}

```

### 数据摘要 MessageDigest 

`MessageDigest` 提供了数据摘要的功能，它可以计算任意大小的数据，并得到一个固定长度的结果。一般来说，它能够保证不同数据计算出的结果是不一致，能保证无法通过结果逆向出数据。所以 `MessageDigest` 的计算结果可以看做是数据的指纹，用于判断数据是否被更改。

BouncyCastle 提供的数据摘要算法包括：

| Name     | Output (bits) | Notes      |
| :------- | :------------ | :--------- |
| MD2      | 128           |            |
| MD4      | 128           |            |
| MD5      | 128           |            |
| SHA1     | 160           |            |
| SHA-224  | 224           | FIPS 180-2 |
| SHA-256  | 256           | FIPS 180-2 |
| SHA-384  | 384           | FIPS 180-2 |
| SHA-512  | 512           | FIPS 180-2 |
| SHA3-224 | 224           |            |
| SHA3-256 | 256           |            |
| SHA3-384 | 384           |            |
| SHA3-512 | 512           |            |
| SM3      | 256           |            |

```java
import org.junit.Assert;
import org.junit.Test;

import java.security.MessageDigest;

public class MessageDigestTest {

    @Test
    public void digest() throws Exception {
        MessageDigest digest = MessageDigest.getInstance("SHA-256", ProviderUtils.PROVIDER);
        digest.update("data 1".getBytes());
        digest.update("data 2".getBytes());
        byte[] bytes = digest.digest();
        Assert.assertEquals(32, bytes.length);
    }

}
```

### 消息认证 Mac

