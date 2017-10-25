---
title: RSA/AES加密
date: 2017-10-25 16:43:34
tags: [加密]
categories: Android代码库
---
RSA/AES加密工具类
<!-- more -->

## RSA
```java
import sun.security.rsa.RSAPrivateCrtKeyImpl;
import sun.security.rsa.RSAPublicKeyImpl;

import javax.crypto.Cipher;
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.security.*;
import java.util.Base64;

public class RSAUtil {


    private static final String ALGORITHM = "RSA";


    //生成密钥对
    public static KeyPair genKeyPair(int keyLength) throws NoSuchAlgorithmException {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance(ALGORITHM);
        keyPairGenerator.initialize(keyLength);
        return keyPairGenerator.generateKeyPair();
    }


    //以base64编码保存密钥
    public static void saveKey(Key key, String path) throws IOException {
        Path keyPath = Paths.get(path);
        Files.write(keyPath, Base64.getEncoder().encode(key.getEncoded()));
    }

    //读取公钥
    public static PublicKey readPublicKey(String path) throws Exception {
        Path keyPath = Paths.get(path);
        byte[] keyBytes = Files.readAllBytes(keyPath);
        return new RSAPublicKeyImpl(Base64.getDecoder().decode(keyBytes));
    }

    //读取密钥
    public static PrivateKey readPrivateKey(String path) throws Exception {
        Path keyPath = Paths.get(path);
        byte[] keyBytes = Files.readAllBytes(keyPath);
        return RSAPrivateCrtKeyImpl.newKey(Base64.getDecoder().decode(keyBytes));
    }


    //公钥加密
    public static String encrypt(String content, PublicKey publicKey) throws Exception {
        //选择算法,创建实例
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        //选择模式,结合公钥初始化
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        //加密
        byte[] result = cipher.doFinal(content.getBytes());
        //转码
        String base64Result = Base64.getEncoder().encodeToString(result);
        return base64Result;
    }

    //私钥解密
    public static String decrypt(String content, PrivateKey privateKey) throws Exception {
        //创建实例
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        //初始化
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        //转码
        byte[] encodedBytes = Base64.getDecoder().decode(content.getBytes());
        //解密
        byte[] result = cipher.doFinal(encodedBytes);
        return new String(result);
    }


    public static void main(String[] args) {

        try {

            //确保目录存在
            File f = new File("/home/duoyi/encrypt/");
            f.mkdirs();


            KeyPair keyPair = RSAUtil.genKeyPair(1024);

            //获取公钥
            PublicKey publicKey = keyPair.getPublic();
            System.out.println("公钥：" + new String(Base64.getEncoder().encode(publicKey.getEncoded())));

            //获取私钥
            PrivateKey privateKey = keyPair.getPrivate();
            System.out.println("私钥：" + new String(Base64.getEncoder().encode(privateKey.getEncoded())));

            //保存密钥
            RSAUtil.saveKey(publicKey, "/home/duoyi/encrypt/rsa.pub");
            RSAUtil.saveKey(privateKey, "/home/duoyi/encrypt/rsa.pri");


            String content = "this is content";


            //读取密钥
            PublicKey pubKey = RSAUtil.readPublicKey("/home/duoyi/encrypt/rsa.pub");
            PrivateKey priKey = RSAUtil.readPrivateKey("/home/duoyi/encrypt/rsa.pri");

            //加密
            String encryptBase64 = RSAUtil.encrypt(content, pubKey);

            //解密
            String result = decrypt(encryptBase64, priKey);

            System.out.println("密文为:" + new String(Base64.getDecoder().decode(encryptBase64.getBytes())));
            System.out.println("密文转码后为:" + encryptBase64);
            System.out.println("转码后解码为:" + result);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}

```
输出
```
公钥：MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCwX7B6S8teU8b7YXgaT9kOoJXP6GHrPnrV9VNQUrKYP701dsIVzWuTCxqp99JJTtRb6KRhzpTdUkwG4bWLVq5mhBb5ItglV6sU9bXQdx8Lc69xKQ0+5W6WvGYfosQEdv/h3eKvQEPbs5rGfoTka4u3Gh4Ncf9McYzI6O5E+eHQRQIDAQAB
私钥：MIICdgIBADANBgkqhkiG9w0BAQEFAASCAmAwggJcAgEAAoGBALBfsHpLy15TxvtheBpP2Q6glc/oYes+etX1U1BSspg/vTV2whXNa5MLGqn30klO1FvopGHOlN1STAbhtYtWrmaEFvki2CVXqxT1tdB3Hwtzr3EpDT7lbpa8Zh+ixAR2/+Hd4q9AQ9uzmsZ+hORri7caHg1x/0xxjMjo7kT54dBFAgMBAAECgYBfg4KLyC4bKB1zFya2gRVYAYj/7aXRgqV85v02W4KSRrpNkMGskvE10WagMy/zOThxiXwz527gqGe5tlPdYJTS03zZlgky1IwL80ZN6F5CIc2LzDTkbDHXHIG5kMt4OOZf/UtM4W+lCNGt4azt5sWlIOoWE6vXQOLfv1eUiQfsAQJBANdThIeP+O4kn0MA2QWkcV39v4ZBnftK4wsXvrFOhMBceiSqJCPwpU2/Tf65JO0KH3lEo/s+ZUjctz2SFX/yc0UCQQDRsI6And20aZi4+SaRnK/4BZIWbklchdkrWloWRj0Rp/NBi9p4lX9ZCNQs0zPDo8pRY2Qvhlj7yuPUHnMPATkBAkEAu/5c3QZj7Xbn3VXmJDj4CXm7N3oedgFhzJOEl8TXviJ/OXeaag52JDT74YK/rHyEEhpNmNNXFpAtI4JhZv3EiQJAPgvAHs6Xi4qzZghTIUL7zqfXUkvP6VCxseJKRc0CxPatQ/fd7VBPHkk+fwT/jCQq+WoveuCF8/tU7q8T3JzAAQJAOXPzNmxQaD+Jt7I8nbJdU70lpRkuU2dwhmOG9DS8ICwo2JGk5woXbUfGTA7DW5K9MMb3oz9cjWV0Dj+tLJicYA==
�nL��߫�0�GR�mĨ������������m+��9u�}0��5H�_B��9�'�"vU������ކ?~_f�ϭ˪=��ʩ�/�O�T��q��"�jn4�}
密文转码后为:ORHTTIVB8bRDeFGU5SZNyELbU32ADdduTJvs36vwMLUZR1KVbcSo574Qxx3wzBu+tQSJ0dn4pcVtK6T4OXXSHRd9MP4f+TVImV9C/LoLOeMnyCIbdlW2j6TIw/Xehj9+X2bez63Lqj3D68qpuC/pT+EHVK+icfwZ5iL+am405H0=
转码后解码为:this is content
```

## AES
```java
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.security.Key;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.util.Base64;
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;


public class AESUtil {

    public static final String ALGORITHM = "AES";
    public static final String RANDOM_ALGORITHM = "SHA1PRNG";


    //选择算法,根据密码生成密钥
    public static SecretKey genKey(String password) throws NoSuchAlgorithmException {
        KeyGenerator keyGenerator = KeyGenerator.getInstance(ALGORITHM);
        SecureRandom random = SecureRandom.getInstance(RANDOM_ALGORITHM);
        random.setSeed(password.getBytes());//设置密钥
        keyGenerator.init(random);
        return keyGenerator.generateKey();
    }

    //以base64编码保存密钥
    public static void saveKey(Key key, String path) throws IOException {
        Path keyPath = Paths.get(path);
        Files.write(keyPath, Base64.getEncoder().encode(key.getEncoded()));
    }

    //读取密钥
    public static SecretKey readSecretKey(String path) throws Exception {
        Path keyPath = Paths.get(path);
        byte[] keyBytes = Files.readAllBytes(keyPath);
        return new SecretKeySpec(Base64.getDecoder().decode(keyBytes), ALGORITHM);
    }


    public static String encrypt(String content, SecretKey secretKey) throws Exception {
        //指定算法创建Cipher实例
        Cipher cipher = Cipher.getInstance(ALGORITHM);//算法是AES
        //选择模式，结合密钥初始化Cipher实例
        cipher.init(Cipher.ENCRYPT_MODE, secretKey);
        //加密
        byte[] result = cipher.doFinal(content.getBytes());
        //使用Base64对密文进行转码
        String base64Result = Base64.getEncoder().encodeToString(result);
        return base64Result;
    }


    public static String decrpyt(String content, SecretKey secretKey) throws Exception {
        //获取实例
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        //初始化
        cipher.init(Cipher.DECRYPT_MODE, secretKey);
        //转码
        byte[] encodedBytes = Base64.getDecoder().decode(content.getBytes());
        //解密
        byte[] result = cipher.doFinal(encodedBytes);
        return new String(result);
    }

    public static void main(String[] args) {
        try {
            //确保目录存在
            File f = new File("/home/duoyi/encrypt/");
            f.mkdirs();

            String content = "this is content";
            String password = "passwd";

            SecretKey key = AESUtil.genKey(password);

            AESUtil.saveKey(key, "/home/duoyi/encrypt/aes.key");

            String encryptBase64 = AESUtil.encrypt(content, key);

            SecretKey readKey = AESUtil.readSecretKey("/home/duoyi/encrypt/aes.key");
            String result = AESUtil.decrpyt(encryptBase64, readKey);

            System.out.println("密文为:" + new String(Base64.getDecoder().decode(encryptBase64.getBytes())));
            System.out.println("密文转码后为:" + encryptBase64);
            System.out.println("转码后解码为:" + result);

        } catch (Exception e) {
            e.printStackTrace();

        }
    }

}
```
输出
```
密文为:����������Fq<
密文转码后为:/hKckfWZxuqJ467Rx0ZxPA==
转码后解码为:this is content
```