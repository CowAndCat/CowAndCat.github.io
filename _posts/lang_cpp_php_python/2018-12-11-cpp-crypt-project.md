---
layout: post
title: 一套跨语言的高速加解密算法
category: c_cpp
comments: false
---

## 一、问题背景

有个算法服务需要私有化部署，其中的模型文件很重要，不允许客户拿到。

现在提供一个能解密文件的USB工具，解密文件大小上限为32KB。工具不能存储内容，只能用来加解密。工具可以复制（两个工具可以做同样的事情，相当于一把钥匙能配置很多把）。工具的制作和用工具加密文件都在用户隔离网段进行。

基于这个硬件来实现一套方案来保证模型文件的安全。

## 二、方案设计

### 2.1 方案流程
本方案基于RSA和AES来实现文件的加解密。

![加解密方案](/images/201812/dongle_scheme.png "dongle")

1. 首先生成RSA公钥rsa_pub和私钥rsa_pri，私钥rsa_pri用USB加密成License_enc文件发布给客户。
2. 然后基于RSA私钥rsa_pri生成AES密钥aes_key。
3. 用RSA公钥rsa_pub和AES密钥aes_key对模型文件加密。(已实现成一个加密工具)
4. 程序内部用USB解密License_enc得到私钥rsa_pri，再用私钥得到AES密钥aes_key。（已封装成动态库，直接调用接口即可）
5. 用RSA私钥rsa_pri和AES密钥aes_key解密模型文件内容到内存。
6. 在服务运行完毕（或中间存储）时，重复4，再用RSA私钥rsa_pri和AES密钥aes_key加密内存到模型文件。

用到的加密算法及其特性。

|加密算法|算法类型|密钥长度|可逆性|运算速度|安全性|资源消耗|
|--|--|--|--|--|--|--|
|AES|对称加密（加解密密钥相同）|128bit|密钥加解密|快|较高|低|
|RSA|非对称加密（加解密密钥不同）|1024bit|公钥加密，私钥加解密|慢|高|高|
|MD5|散列算法|不限(产出32B长度)|不可逆|快|中|低|

### 2.2 加解密算法策略
如下图：

![算法策略](/images/201812/crypt_strategy.png "crypto")

1.为什么要用混合双打算法？  
首先RSA算法是必须，因为只有它能保证安全性。但是它贼慢，1024bit的公私密钥每次只能加密至多117B数据，加密10M内容要8秒。
所以策略上用另一种对称算法加密后续的内容。这样恶意用户即便知道对称加密密钥，也得不到完整的明文。如果不知道本加密策略，连不完整的明文都拿不到。

2.128个RSA加密块，这个数值是怎么确定的？  
TradeOff，这个块数和加密速度就像是在一个跷跷板的两端，可以加大块数，但是得牺牲加解密速度。对一个文件的前15KB数据进行RSA加密，能适应大部分场景了。

3.为什么要把AES加密的block_cnt和last_block_len也存储到密文里？  
AES每次加解密的内容必须是16的整数倍（当密钥是128b长度时），如果明文长度不是，必须补齐。这就给解密增加了难度：解密的时候并不知道原来明文的长度。（用`\0`补齐？这并不适用于二进制的模型文件，二进制文件里什么鬼都有）  
block_cnt这个字段用于表示最后block位置，如果解密到这个block，memset明文的缓存大小就不用10M了，可以直接用last_block_len来设置，可以节省点空间。（解密时，补齐的内容直接被废弃掉了，因为只memset那么多空间）
所以需要用这两个字段来协助解密。


## 2.3 方案自评（chui niu bi）

- 安全。如果缺少了License_enc文件或USB硬件，对模型不能进行加解密。
- 熟练掌握了RSA和AES加密算法的特性，并扬长避短地集成了算法的优点。(RSA的安全和AES的速度)
- 设计巧妙，每次加密后的文件内容都会不同！
- 设计巧妙，能支持的文件大小上限为：2147483648\*10 MB。
- 设计巧妙，License_enc文件和加密工具都可以公开，通过网络传输给客户。

不足：

- 复杂

## 三、方案实现

算法借助于openssl算法库。

加解密算法源码如下，仅供学习使用，蟹蟹。  
crypt_util.h:

    #ifndef CRYPT_UTIL_H
    #define CRYPT_UTIL_H

    #include <string>
    #include <string.h>

    using namespace std;

    namespace crypt_util {
        int decrypt(const string &private_key, const string &src_str, string &dest_str);

        int encrypt(const string &private_key, const string &src_str, string &dest_str);
    }
    #endif //CRYPT_UTIL_H

crypt_util.cpp:

    #include <iostream>
    #include <string>
    #include <stdio.h>
    #include <stdlib.h>
    #include "openssl/rsa.h"
    #include "openssl/pem.h"
    #include "openssl/md5.h"
    #include "openssl/aes.h"
    #include "crypt_util.h"

    namespace crypt_util {
        const int BLOCK_NUM_RSA = 128;
        const int BLOCK_SIZE_RSA = 115;
        const int BLOCK_SIZE_AES = 10 * 1024 * 1024;

        inline int get_md5(const string &src_str, string &dest_src) {
            unsigned char md5_str[33] = {0};
            MD5((const unsigned char *) src_str.c_str(), src_str.length(), md5_str);
            char buf[65] = {0};
            char tmp[3] = {0};
            for (int i = 0; i < 32; i++) {
                sprintf(tmp, "%02x", md5_str[i]);
                strcat(buf, tmp);
            }
            buf[32] = '\0';
            dest_src = std::string(buf);
            return 1;
        }

        inline int chars_to_int(const unsigned char *chars) {
            int addr = chars[0];
            addr |= (chars[1] << 8);
            addr |= (chars[2] << 16);
            addr |= (chars[3] << 24);
            return addr;
        }

        inline void int_to_chars(int i, char *chars) {
            memset(chars, 0, sizeof(char) * 4);
            chars[0] = (char) (0xff & i);
            chars[1] = (char) ((0xff00 & i) >> 8);
            chars[2] = (char) ((0xff0000 & i) >> 16);
            chars[3] = (char) ((0xff000000 & i) >> 24);
        }

        void save_aes_block_info(int src_len, int pos, string &dest_str) {
            // calculate block count and the len of last block.
            int aes_block_count = (src_len - pos) / BLOCK_SIZE_AES;
            int last_block_len = (src_len - pos) % BLOCK_SIZE_AES;
            if (last_block_len != 0) {
                aes_block_count = aes_block_count + 1;
            }
            char chars[4];
            int_to_chars(aes_block_count, chars);
            dest_str.append(string(chars, 4));
            int_to_chars(last_block_len, chars);
            dest_str.append(string(chars, 4));
        }

        inline int align_to_16s(int len, string &sub_src_str) {
            if (len % 16 != 0) {
                len = len + (16 - len % 16);
                for (int index = 0; index < 16 - (len % 16); index++) {
                    sub_src_str.append("\0");
                }
            }
            return len;
        }

        int init_keys(const string &private_key, RSA *&rsa, BIO *&key_bio, AES_KEY &aes_key, int type = 0) {
            rsa = RSA_new();
            key_bio = BIO_new_mem_buf((unsigned char *) private_key.c_str(), -1);
            rsa = PEM_read_bio_RSAPrivateKey(key_bio, &rsa, NULL, NULL);
            if (rsa == NULL) {
                fprintf(stderr, "Failed to create RSA.\n");
                return -1;
            }
            // gen the 16-bytes AES key from RSA private_key.
            string aes_key_str;
            get_md5(private_key, aes_key_str);
            aes_key_str = aes_key_str.substr(0, 16);
            unsigned char aes_key_arr[AES_BLOCK_SIZE + 1];
            memset(aes_key_arr, 0x00, sizeof(aes_key_arr));
            memcpy(aes_key_arr, aes_key_str.c_str(), AES_BLOCK_SIZE);
            memset(&aes_key, 0x00, sizeof(AES_KEY));
            if (type == 1) {
                if (AES_set_encrypt_key(aes_key_arr, 128, &aes_key) < 0) {
                    fprintf(stderr, "Unable to set encryption key in AES...\n");
                    return -1;
                }
            } else {
                if (AES_set_decrypt_key(aes_key_arr, 128, &aes_key) < 0) {
                    fprintf(stderr, "Unable to set decryption key in AES...\n");
                    return -1;
                }
            }
            return 0;
        }

        int encrypt(const string &private_key, const string &src_str, string &dest_str) {
            RSA *rsa;
            BIO *key_bio;
            AES_KEY aes_key;
            int ret = init_keys(private_key, rsa, key_bio, aes_key, 1);
            if (ret == -1) {
                fprintf(stderr, "Unable to init keys.\n");
                return -1;
            }
            unsigned char ivec[AES_BLOCK_SIZE];
            for (int i = 0; i < AES_BLOCK_SIZE; ++i) {
                ivec[i] = 0;
            }

            size_t pos = 0;
            char *encrypted_text;
            int block_count = 0;
            int len = BLOCK_SIZE_RSA;
            const int rsa_len = RSA_size(rsa);
            string sub_src_str = src_str.substr(pos, (size_t) len);
            while (!sub_src_str.empty()) {
                block_count++;
                pos = pos + len;
                if (block_count <= BLOCK_NUM_RSA) {
                    encrypted_text = (char *) malloc((size_t) rsa_len + 1);
                    memset(encrypted_text, 0, (size_t) rsa_len + 1);
                    // Must use the ret as length when appending encrypted_text to dest_str.
                    ret = RSA_public_encrypt((int) sub_src_str.length(), (const unsigned char *) sub_src_str.c_str(),
                            (unsigned char *) encrypted_text, rsa, RSA_PKCS1_PADDING);
                } else {
                    if (len % 16 != 0) {
                        len = len + (16 - len % 16);
                        for (int iii = 0; iii < 16 - len % 16; iii++) {
                            sub_src_str.append("\0");
                        }
                    }
                    ret = len;
                    encrypted_text = (char *) malloc((size_t) ret + 1);
                    memset(encrypted_text, 0, (size_t) ret + 1);
                    AES_cbc_encrypt((unsigned char *) sub_src_str.c_str(), (unsigned char *) encrypted_text,
                            ret, &aes_key, ivec, AES_ENCRYPT);
                }
                dest_str.append(string(encrypted_text, ret));
                free(encrypted_text);
                if (pos > src_str.length()) {
                    break;
                }
                if (block_count == BLOCK_NUM_RSA) {
                    // Turn to AES encrypt after block_count.
                    save_aes_block_info(src_str.length(), pos, dest_str);
                    len = BLOCK_SIZE_AES;
                }
                sub_src_str = src_str.substr(pos, (size_t) len);
                len = (int) sub_src_str.length();
            }

            BIO_free_all(key_bio);
            RSA_free(rsa);
            return 0;
        }

        int decrypt(const string &private_key, const string &src_str, string &dest_str) {
            RSA *rsa;
            BIO *key_bio;
            AES_KEY aes_key;
            int ret = init_keys(private_key, rsa, key_bio, aes_key);
            if (ret == -1) {
                fprintf(stderr, "Unable to init keys.\n");
                return -1;
            }
            unsigned char ivec[AES_BLOCK_SIZE];
            for (int i = 0; i < AES_BLOCK_SIZE; ++i) {
                ivec[i] = 0;
            }

            int block_count = 0;
            size_t pos = 0;
            int len = RSA_size(rsa);
            string sub_src_str = src_str.substr(pos, (size_t) len);
            char *decryptedText;
            int aes_block_count = 0;
            int aes_block_index = 1;
            int last_block_len = 0;

            while (!sub_src_str.empty()) {
                block_count++;
                pos += len;
                if (block_count <= BLOCK_NUM_RSA) {
                    decryptedText = (char *) malloc((size_t) len + 1);
                    memset(decryptedText, 0, (size_t) len + 1);
                    ret = RSA_private_decrypt(len, (const unsigned char *) sub_src_str.c_str(),
                            (unsigned char *) decryptedText, rsa, RSA_PKCS1_PADDING);
                } else {
                    ret = len;
                    if (aes_block_index >= aes_block_count) {
                        ret = last_block_len;
                    }
                    aes_block_index++;
                    decryptedText = (char *) malloc((size_t) ret + 1);
                    memset(decryptedText, 0, (size_t) ret + 1);
                    AES_cbc_encrypt((unsigned char *) sub_src_str.c_str(), (unsigned char *) decryptedText,
                            ret, &aes_key, ivec, AES_DECRYPT);
                }
                dest_str.append(string(decryptedText, (size_t) ret));
                free(decryptedText);

                if (pos > src_str.length()) {
                    break;
                }
                if (block_count == BLOCK_NUM_RSA) {
                    // Turn to AES decrypt after block_count.
                    // get block count and the len of last block from 8-bytes.
                    string int_chars = src_str.substr(pos, 4);
                    aes_block_count = chars_to_int((unsigned char *) int_chars.c_str());
                    int_chars = src_str.substr(pos + 4, 4);
                    last_block_len = chars_to_int((unsigned char *) int_chars.c_str());
                    pos = pos + 8;
                    len = BLOCK_SIZE_AES;
                }
                sub_src_str = src_str.substr(pos, (size_t) len);
            }
            BIO_free_all(key_bio);
            RSA_free(rsa);
            return 0;
        }
    }

用硬件解密License_enc的代码省略。

加密工具Python实现（这个可以优化成C++代码，并将之编译成二进制文件发布）：
    
    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
    """
    File encryptor.
    Usage:
        python3 demo_enc.py TO_ENCRYPT_FILE ENCRYPTED_FILE
    """
    from Crypto.PublicKey import RSA
    from Crypto.Cipher import PKCS1_v1_5 as Cipher_pkcs1_v1_5
    from Crypto.Cipher import AES

    import os
    import sys
    import argparse

    aes_key = 'e27283e6fc0f8a4c'
    BLOCK_NUM = 128
    AES_BLOCK_SIZE = 10 * 1024 * 1024


    def get_args():
        """
        get arguments.
        :return: args.
        """
        parser = argparse.ArgumentParser()
        parser.add_argument("from_file", nargs=1, help=" file to decrypt.", type=str)
        parser.add_argument("to_file", nargs=1, help=" the file to save encrypted content.",
            type=argparse.FileType('wb'), default=sys.stdout)
        return parser.parse_args()


    def int_to_bytes(num):
        """int to bytes"""
        return bytes([0xff & num, (0xff00 & num) >> 8, (0xff0000 & num) >> 16, (0xff000000 & num) >> 24])


    def encrypt_file(from_file_path, to_file, length=115):
        """
        encrypt file.
        :param from_file:
        :param to_file:
        :param length:
        :return:
        """
        rsa_key = RSA.importKey('-----BEGIN PUBLIC KEY-----\nMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC2MR/PvDR/LZReFc4gLWLcQVjl\nopOjn9PLG464vFrGTlvfaC7vA9t5Kex608pLTbgr06XrXj8zBXvnuQ2S9rDHVzRd\n54na+NQqot6hLXAErSStqcAB243En2E8L11o9GrNoLHiNfvOnwN9kX3VS4lB+F22\nqvrVUDpEGc9kZ0zZTwIDAQAB\n-----END PUBLIC KEY-----')
        rsa_cipher = Cipher_pkcs1_v1_5.new(rsa_key)
        aes_cipher = AES.new(aes_key, AES.MODE_CBC, (b'\0' * 16))
        total_len = os.path.getsize(from_file_path)
        pos = 0
        with open(from_file_path, 'rb') as from_file:
            text = from_file.read(length)
            count = 0
            while text:
                pos += length
                count += 1
                if count <= BLOCK_NUM:
                    cipher_text = rsa_cipher.encrypt(text)
                else:
                    cipher_text = aes_encrypt(aes_cipher, text)
                to_file.write(cipher_text)
                if count == BLOCK_NUM:
                    # Begin to encrypt with AES cipher after RSA cipher, then the speed will take off.
                    # write aes block info.
                    aes_block_count = int((total_len-pos)/AES_BLOCK_SIZE)
                    last_block_len = (total_len-pos)%AES_BLOCK_SIZE
                    if last_block_len!=0:
                        aes_block_count = aes_block_count+1;
                    to_file.write(int_to_bytes(aes_block_count))
                    to_file.write(int_to_bytes(last_block_len))
                    length = AES_BLOCK_SIZE
                text = from_file.read(length)


    def aes_encrypt(cipher, src_str):
        """ AES encrypt. """
        length = 16
        count = len(src_str)
        if count % length != 0:
            add = length - (count % length)
        else:
            add = 0
        text = src_str + (b'\0' * add)
        return cipher.encrypt(text)


    if __name__ == '__main__':
        args = get_args()
        encrypt_file(args.from_file[0], args.to_file[0])

LicenseCenter部分代码：

    import os
    import hashlib
    import logging
    from Crypto import Random
    from Crypto.PublicKey import RSA

    ENCODING = 'UTF-8'
    PRIVATE_KEY_DIR = '.rsa'
    MACRO_PUBLIC_KEY = '{{RSA_PUBLIC}}'
    MACRO_AES_KEY = '{{AES_KEY}}'
    ENC_TEMPLATE = '../Encryptor_template.py'


    def get_signature(service_name):
        """Get the signature of service."""
        return hashlib.md5(bytes(service_name + 'some_salt', ENCODING)).hexdigest()


    class LicenseCreator(object):
        """License Creator"""

        def __init__(self, service_name,  update_key=False):
            self.service_name = service_name
            self.rsa_pri_file = '.%s_rsa' % service_name
            self.encrypt_file = '%s_enc.py' % service_name
            self.update_key = update_key

        def gen_license_files(self, directory=None):
            """

            :param directory:
            :return:
            """
            if directory is None:
                directory = self.service_name
            if not os.path.exists(directory):
                os.makedirs(directory)
            rsa_public, rsa_private = self._gen_rsa(self.rsa_pri_file)
            os.chdir(directory)
            self._gen_encryptor(rsa_public, rsa_private)
            plain_license_file = self.service_name
            with open(plain_license_file, 'w') as f:
                f.write(self._gen_plain_license(rsa_private))
            os.system('Some CMD: use the USB to encrypt plain_license_file' % (plain_license_file))
            os.remove(plain_license_file)

        def _gen_plain_license(self, private_key):
            return private_key.decode(ENCODING)

        def _gen_encryptor(self, rsa_public, rsa_private):
            """Generate encryptor."""
            rsa_public = rsa_public.decode(ENCODING).replace('\n', '\\n')
            aes_key = hashlib.md5(rsa_private).hexdigest()[:16]
            with open(ENC_TEMPLATE, 'r') as f:
                with open(self.encrypt_file, 'w') as w:
                    for line in f.readlines():
                        w.write(line.replace(MACRO_PUBLIC_KEY, rsa_public)
                            .replace(MACRO_AES_KEY, aes_key))
            logging.info('Generate encrytor successfully.')

        def _gen_rsa(self, private_key_file):
            """Generate rsa key pair."""
            if not os.path.exists(PRIVATE_KEY_DIR):
                os.mkdir(PRIVATE_KEY_DIR)
            cur_dir = os.getcwd()
            os.chdir(PRIVATE_KEY_DIR)
            if not self.update_key and os.path.exists(private_key_file):
                with open(private_key_file, 'rb') as f:
                    rsa = RSA.importKey(f.read())
            else:
                rsa = RSA.generate(1024, Random.new().read)
                with open(private_key_file, 'wb') as f:
                    f.write(rsa.exportKey())
            os.chdir(cur_dir)
            return rsa.publickey().exportKey(), rsa.exportKey()

## 四、其他

### 4.1 编译动态库

编译libdongle.so的命令：

    g++ crypt_util.cpp -o libdongle.so -shared -fPIC -ldl -lcrypto -std=c++11

### 4.2 加密工具使用

加密文件file1,存储到file2：

    python3 demo_enc.py file1 file2

### 4.3 吐槽

- 说是跨语言的方案，也就只涉及C++和Python嘛，而且Python那部分完全可以用C++来写呀！（是是是，有空再改，有空再改）
- 算法的门槛已经有三四层楼那么高了，就算让客户拿到一个模型文件，他们能干啥！？
