---
title: mbedtls加解密库
date: 2019-05-21 16:26:28
tags: [Embedded, 加密]
categories:
- 嵌入式
- 加密
comments: true
---

开源项目`mbedtls`，是一个专用于嵌入式arm平台的加解密方案，包含很多算法

github仓库地址[https://github.com/ARMmbed/mbedtls](https://github.com/ARMmbed/mbedtls)
官网地址[https://tls.mbed.org/](https://tls.mbed.org/)

## AES-CBC 算法测试
AES有5种加密方式
* 电码本模式(Electronic Codebook Boo, ECB)
* 密码分组链接模式(Cipher Block Chaining, CBC)
* 计算器模式(Counter, CTR)
* 密码反馈模式(Cipher FeedBack, CFB)
* 输出反馈模式(Output FeedBack, OFB)

AES-CBC算法API使用的官方文档链接[https://tls.mbed.org/kb/how-to/encrypt-with-aes-cbc](https://tls.mbed.org/kb/how-to/encrypt-with-aes-cbc)，

我将`mbedtls`源码在本地下载安装后，选择AES-CBC算法编写了测试程序，该算法密钥长度可为128bit、192bit或256bit，代码如下
```c
#include 'aes.h'

#define KEY_LEN 16  // 定义密钥长度
#define ALIGN_AES(input) (((strlen(input)>>4)+1)<<4)    // 以密钥长度对齐，分组加密时输入长度需要以key对齐

int main(int argc, char **argv)
{
    mbedtls_aes_context aes_ctx;    // aes密钥管理结构，密钥放在其中

    unsigned char key[KEY_LEN] = {  // 设置密钥的值
		0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x08,0x09,0x0A,0x0B,0x0C,0x0D,0x0E,0x0F,0x00
	};

    unsigned char enc_iv[16]={0x1}; // 初始向量
    unsigned char dec_iv[16]={0x1};

    unsigned char plain[16] = "hello world";    // 明文
    unsigned char dec_plain[16] = {0x0};        // 解密后的明文
    unsigned char cipher[16] = {0x0};           // 密文

    log_info("plain: [ %s ], len %d", plain, strlen(plain));    // 打印明文内容
    mbedtls_aes_init(&aes_ctx); // 初始化加密结构
    mbedtls_aes_setkey_enc(&aes_ctx, key, 128);     // 设置加密密钥

    mbedtls_aes_crypt_cbc(&aes_ctx, MBEDTLS_AES_ENCRYPT, ALIGN_AES(plain),  // 加密
        enc_iv, plain, cipher);     
    log_info("cipher [ %s ] len %d", cipher, strlen(cipher));    // 打印密文内容


    mbedtls_aes_setkey_dec(&aes_ctx, key, 128);     // 设置解密密钥
    mbedtls_aes_crypt_cbc(&aes_ctx, MBEDTLS_AES_DECRYPT, ALIGN_AES(cipher), // 解密
        dec_iv, cipher, dec_plain);
    log_info("dec_plain [ %s ] len %d", dec_plain, strlen(dec_plain));  // 打印解密后的明文

    mbedtls_aes_free(&aes_ctx); // 释放
    return 0;
}
```

运行程序，打印内容如下
```shell
[root@localhost aes_test]# ./bin/aes_test
 [2019/05/22 00:59:34] [INFO] [main:714 ] plain: [ hello world ], len 11
 [2019/05/22 00:59:34] [INFO] [main:715 ]

 [2019/05/22 00:59:34] [INFO] [main:719 ] cipher [ Z!] len 16
 [2019/05/22 00:59:34] [INFO] [main:720 ]

 [2019/05/22 00:59:34] [INFO] [main:724 ] dec_plain [ hello world ] len 11
 [2019/05/22 00:59:34] [INFO] [main:725 ]

[root@localhost aes_test]#
```

明文`hello world`被正确的解密

为了模拟海思平台的发包内容，设置收发`buff`长度为489，模拟业务数据；数据内容以0~100的随机整数填充，测试结果如下
```shell
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> plain byte:
13 19 46  c 63 57  2 28 3e 2f 31 5c 21 35 1e  b
25  0  d  8 3c 36 19 60 54 46 3c 38 13 17 52 26
31 34  2  0 27 38 29 36  4 2a 62 59 5f 1c 34 21
1c 12 5d 28 48 46 58  8 29 31 11 3c 48 63 32 49
 3 34 4a 5f 3c 43 31 40  9 2f  6 38 4b 3a 29 38
4c 23 60  0  5 55  9 62 22 1a  a 3a 4d 3c 54 50
40 3a 4b 19 4d 4c 29 56 4b 63 5e  3 3a 24 3b 56
17  7 57 50 5c 30 1f 4e 1a 29 25 37 36 15 23 46
1f 3f 2f  8 5b 29 2e 43 28 28 46 32 1c 51 25 33
58 4c 54 21 4c  f  b 36 38  0  9  a 49 60 51  4
3b 50 40  3 15  a 46 3e  3 5c  c 1f 19  1 53  d
4d 13 2e  5 22  a 3b 5a  a 14 35 24 11 56 28 1c
42 39 1f 58 43 35  2 46 2d  e 36 46 44 59 24 2d
 8 22  3 2a 2c 3e 54  7 53 25 2b  0 17 23 50 2a
2c  c 1e 40 41 54 22 3f 62 28 55 12 1d 49 40 25
 8 43 4f  4 51 40 3f 10 35 3a 44 4d 2e 31 47 5a
3d  1 36 4e 25 29 29 23 51 4f  6  b 34 46 30 3c
59 50 11 16 60 50 27 31 27  7 4e 55 38 31 4b 45
 2 52  0 27 17 5d 1b  4 48 21 43 19 37 10 25 60
30 36 12 2c 23  9 2d 1a 45 18 3f 4d 19 5a 2f 1c
48 63 13 5f 5c 2e 34 41 1f 13 5a 26 57 1b 22 23
22  5 1f 45  e 4d 2f 23 35  a  d 4e  0  c 3a 49
 b 1e 14  3 1c 48 44 3c 5c  a 62 4f 26 55 43 18
5a 62 2d 38 1b 5c 5c 50 36  5  b 36 11 15 4f 1c
33  0 53 50 18 34 5c 10 3e 2a 30 34 1b  f 4c 11
 d 15 1a 5d 41 12 49 13 4b 24 1a 2c  a 39 18  d
 9  7 2d 22  b 25  2 1a 50 32 1e 3b 41  7 1d 1f
50 37 18 62 19 31 11 34 26 5f 60 30 35 48 3d 3e
1f 3b 30 2b 60 33 45 1c  1 63 58 13 3a 45 32 5b
18 1a 29  1 1b  a 35 41  6  1 41  b 49 4f 49  4
26 16 63 56 19 14 43 1a 48
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<


>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> dec_plain byte:
13 19 46  c 63 57  2 28 3e 2f 31 5c 21 35 1e  b
25  0  d  8 3c 36 19 60 54 46 3c 38 13 17 52 26
31 34  2  0 27 38 29 36  4 2a 62 59 5f 1c 34 21
1c 12 5d 28 48 46 58  8 29 31 11 3c 48 63 32 49
 3 34 4a 5f 3c 43 31 40  9 2f  6 38 4b 3a 29 38
4c 23 60  0  5 55  9 62 22 1a  a 3a 4d 3c 54 50
40 3a 4b 19 4d 4c 29 56 4b 63 5e  3 3a 24 3b 56
17  7 57 50 5c 30 1f 4e 1a 29 25 37 36 15 23 46
1f 3f 2f  8 5b 29 2e 43 28 28 46 32 1c 51 25 33
58 4c 54 21 4c  f  b 36 38  0  9  a 49 60 51  4
3b 50 40  3 15  a 46 3e  3 5c  c 1f 19  1 53  d
4d 13 2e  5 22  a 3b 5a  a 14 35 24 11 56 28 1c
42 39 1f 58 43 35  2 46 2d  e 36 46 44 59 24 2d
 8 22  3 2a 2c 3e 54  7 53 25 2b  0 17 23 50 2a
2c  c 1e 40 41 54 22 3f 62 28 55 12 1d 49 40 25
 8 43 4f  4 51 40 3f 10 35 3a 44 4d 2e 31 47 5a
3d  1 36 4e 25 29 29 23 51 4f  6  b 34 46 30 3c
59 50 11 16 60 50 27 31 27  7 4e 55 38 31 4b 45
 2 52  0 27 17 5d 1b  4 48 21 43 19 37 10 25 60
30 36 12 2c 23  9 2d 1a 45 18 3f 4d 19 5a 2f 1c
48 63 13 5f 5c 2e 34 41 1f 13 5a 26 57 1b 22 23
22  5 1f 45  e 4d 2f 23 35  a  d 4e  0  c 3a 49
 b 1e 14  3 1c 48 44 3c 5c  a 62 4f 26 55 43 18
5a 62 2d 38 1b 5c 5c 50 36  5  b 36 11 15 4f 1c
33  0 53 50 18 34 5c 10 3e 2a 30 34 1b  f 4c 11
 d 15 1a 5d 41 12 49 13 4b 24 1a 2c  a 39 18  d
 9  7 2d 22  b 25  2 1a 50 32 1e 3b 41  7 1d 1f
50 37 18 62 19 31 11 34 26 5f 60 30 35 48 3d 3e
1f 3b 30 2b 60 33 45 1c  1 63 58 13 3a 45 32 5b
18 1a 29  1 1b  a 35 41  6  1 41  b 49 4f 49  4
26 16 63 56 19 14 43 1a 48
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
```
明文和解密后的明文各字节内容一致，表明解密正确

## 注意
* 加密后的密文长度以密钥的整数倍对齐，会比明文内容稍大
* 收发对端需提前约定好密钥(key)和一个初始向量(iv)
