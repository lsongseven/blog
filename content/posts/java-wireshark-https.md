---
title: "Java中如何用wireshark抓取https"
date: 2020-08-28T22:52:58+08:00
draft: false
---

最近遇到的一个问题是，在更换了https的加密协议后（从ssl更换到tls），之前的post请求总是返回400 bad request这样的问题。网上搜罗了一下，有很多人遇到同样的400 bad request，但是各自的具体情况不尽相同，因此没有找到有效的解决方法。



先说一下遇到的情况，对于某一个post请求，在postman中访问没有问题，但是在java代码中使用httpclient按照原先使用正常的方式去做请求，返回的response为一个null，没有其他具体信息，很难追溯。没有办法，只能一步一步跟进调试，一直跟到org.apache.http.impl.execchain包下MainClientExec.class这个类中，并且可以看到请求是正常发送出去，但是收到的response是null。由于对https的了解不是很足，起初怀疑是不是由于更换到tls后握手出现问题，于是调整org.apache.http这个包下的日志等级为debug，可以看到连接是正常建立，因此排除握手的问题。


```
2020-03-19 16:10:35,119 [org.apache.http.conn.ssl.SSLConnectionSocketFactory.verifyHostname(SSLConnectionSocketFactory.java:423)]-[DEBUG] Secure session established

2020-03-19 16:10:35,119 [org.apache.http.conn.ssl.SSLConnectionSocketFactory.verifyHostname(SSLConnectionSocketFactory.java:424)]-[DEBUG] negotiated protocol: TLSv1.2
2020-03-19 16:10:35,119 [org.apache.http.conn.ssl.SSLConnectionSocketFactory.verifyHostname(SSLConnectionSocketFactory.java:425)]-[DEBUG] negotiated cipher suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
2020-03-19 16:10:35,120 [org.apache.http.conn.ssl.SSLConnectionSocketFactory.verifyHostname(SSLConnectionSocketFactory.java:433)]-[DEBUG] peer principal: CN=demo.bazefield.com, O=Bazefield A/S, L=Porsgrunn, C=NO
2020-03-19 16:10:35,120 [org.apache.http.conn.ssl.SSLConnectionSocketFactory.verifyHostname(SSLConnectionSocketFactory.java:442)]-[DEBUG] peer alternative names: [demo.bazefield.com, www.demo.bazefield.com]
2020-03-19 16:10:35,121 [org.apache.http.conn.ssl.SSLConnectionSocketFactory.verifyHostname(SSLConnectionSocketFactory.java:446)]-[DEBUG] issuer principal: CN=DigiCert SHA2 Secure Server CA, O=DigiCert Inc, C=US
2020-03-19 16:10:35,126 [org.apache.http.impl.conn.DefaultHttpClientConnectionOperator.connect(DefaultHttpClientConnectionOperator.java:145)]-[DEBUG] Connection established 10.77.34.212:52558<->212.33.148.80:443
2020-03-19 16:10:35,126 [org.apache.http.impl.conn.LoggingManagedHttpClientConnection.setSocketTimeout(LoggingManagedHttpClientConnection.java:90)]-[DEBUG] http-outgoing-0: set socket timeout to 3000
main, setSoTimeout(3000) called
2020-03-19 16:10:35,126 [org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:255)]-[DEBUG] Executing request POST /bazefield.services/api/oauth2/token HTTP/1.1
2020-03-19 16:10:35,126 [org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:260)]-[DEBUG] Target auth state: UNCHALLENGED
2020-03-19 16:10:35,127 [org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:266)]-[DEBUG] Proxy auth state: UNCHALLENGED
```


线索断了之后，只能通过抓包的方式看看究竟发生了什么。这里要出现今天讲的主角wireshark，通过这个强大的工具进行抓包，增加ip和ssl的filter （ip.addr==xxx.yy.zzz.80 and ssl）之后可以看到与目标地址的通讯包。但是，由于是https，包中的流量是加密的，并不能进行分析。于是，如何解密抓取的https流量成为关键。



解密之前，首先要了解https的基本流程。这个网上有很多描述详细的文章，这边就不赘述。简单说一下，加密是通过指定的加密方法，结合客户端与服务端之间交换的两个随机数，再加上客户端用服务端提供证书上的public key加密的pre-master这个参数（外界是不知道这个参数的，这个参数由客户端生成，经过公钥加密后，只有服务端知道这个参数是什么），生成一个对称加密密钥。在握手结束后，客户端和服务端之间的通讯流量是采用对称加密算法来加密的，而加密的密钥是有握手阶段交换的随机数random_1，random_2以及pre-master和指定的加密算法生成的。这个过程中，最重要的一个参数就是pre-master，其他的参数都是可以通过抓包得到的。因此，如果能够拿到这个pre-master，那么就可以解密wireshark抓取的https流量。



在前面所说的场景中，我们是客户端，无法知道服务端提供证书的私钥，因此不能从wireshark抓取的包中解密来获得pre-master。但是，我们在代码中所做的请求是通过Apache提供的httpclient完成的，可以在启动时添加vmoption参数-Djavax.net.debug=ssl,keygen来获得tls连接中的一些重要参数。下面是建立连接过程中打印出的部分参数


```
*** ECDH ServerKeyExchange
Signature Algorithm SHA512withRSA
Server key: Sun EC public key, 256 bits
public x coord: 82734322831099238985881665140172758067838155879895358323075425906443044790736
public y coord: 88453306709588242368663096476921278570527038619007749416185523277612481734068
parameters: secp256r1 [NIST P-256, X9.62 prime256v1] (1.2.840.10045.3.1.7)
main, READ: TLSv1.2 Handshake, length = 4
check handshake state: server_hello_done[14]
update handshake state: server_hello_done[14]
upcoming handshake states: client certificate[11](optional)
upcoming handshake states: client_key_exchange[16]
upcoming handshake states: certificate_verify[15](optional)
upcoming handshake states: client change_cipher_spec[-1]
upcoming handshake states: client finished[20]
upcoming handshake states: server change_cipher_spec[-1]
upcoming handshake states: server finished[20]
*** ServerHelloDone
*** ECDHClientKeyExchange
ECDH Public value: { 4, 18, 97, 2, 120, 116, 117, 129, 244, 110, 148, 192, 162, 71, 24, 69, 246, 200, 73, 114, 107, 103, 78, 70, 2, 79, 17, 20, 45, 67, 93, 228, 228, 60, 65, 251, 250, 234, 176, 208, 160, 199, 120, 131, 47, 25, 153, 74, 54, 32, 67, 29, 203, 96, 214, 30, 217, 240, 131, 124, 21, 160, 217, 161, 225 }
update handshake state: client_key_exchange[16]
upcoming handshake states: certificate_verify[15](optional)
upcoming handshake states: client change_cipher_spec[-1]
upcoming handshake states: client finished[20]
upcoming handshake states: server change_cipher_spec[-1]
upcoming handshake states: server finished[20]
main, WRITE: TLSv1.2 Handshake, length = 70
SESSION KEYGEN:
PreMaster Secret:
0000: 1B 26 6E 26 56 BB C2 7B C3 37 7B 3F 75 59 EA AC .&n&V....7.?uY..
0010: 25 53 D1 0E 43 F2 85 49 31 FC 4A 98 C5 56 2C E1 %S..C..I1.J..V,.
CONNECTION KEYGEN:
Client Nonce:
0000: 5E 73 28 FA 0F 1A 88 82 6D 91 5D 50 0E BC 6B 87 ^s(.....m.]P..k.
0010: C5 DD 80 83 AB F7 5E 53 45 CB 34 9B C9 CE 16 02 ......^SE.4.....
Server Nonce:
0000: F1 47 01 E9 55 FE 68 35 6F 43 F4 21 80 30 E9 81 .G..U.h5oC.!.0..
0010: 6D 02 E5 DB B8 67 76 39 60 A3 42 D5 1F 43 32 6C m....gv9`.B..C2l
Master Secret:
0000: 8B E2 88 59 42 7A EE F3 5F EC 25 B0 63 6A FC 91 ...YBz.._.%.cj..
0010: 28 2E A3 70 47 63 19 E0 F5 DB 8D 68 64 77 43 91 (..pGc.....hdwC.
0020: 84 A4 A2 2B 1F 87 91 9D 02 76 EB 55 8A 69 65 BC ...+.....v.U.ie.
... no MAC keys used for this cipher
Client write key:
0000: 44 19 21 EE 46 D5 D3 88 06 C1 6B 4A CF 4D A9 36 D.!.F.....kJ.M.6
0010: 21 BA F1 EE 1A C0 A2 60 C2 3B 18 89 32 A3 59 CA !......`.;..2.Y.
Server write key:
0000: 34 C3 40 A7 20 0F 33 9D 19 2E 02 2D 4C 4E 80 30 4.@. .3....-LN.0
0010: 26 25 31 8A 08 DD D5 F0 F4 86 40 03 56 6A F1 D2 &%1.......@.Vj..
Client write IV:
0000: 54 61 92 79 Ta.y
Server write IV:
0000: 15 62 81 55 .b.U
```


这里我们所要的pre-master就是Clinent Nounce , Master Secret hex值的组合，最后的把它们存在一个sslkey.log文件中，格式如下（手动操作很简单，也可以参考一个python脚本来操作）

CLIENT_RANDOM 5E731E001D49FA112BAF6B0CFFD8EF5C15DF90B27AE6516DFB83FBA484875ECD 1E53B5AFA6925E8E6B594F48C0BCAF64FDCDE60CA2610EBF6693065A4929E096E8287F9172C1D91CCC8134D2DB1370B4

```
##!/usr/bin/env python
import re
import sys


def extract_data_from_line(line):
    m = re.match('\d+:([ 0-9A-F]{51}) .*', line)
    if m:
        return m.group(1).replace(' ', '')
    else:
        raise line


def main():
    f = open("debug.txt", "r")
    parsing_mastersecret_line = 0
    parsing_clientnonce_line = 0
    for line in f.readlines():
        if parsing_mastersecret_line:
            parsing_mastersecret_line += 1
        if parsing_clientnonce_line:
            parsing_clientnonce_line += 1

        if line == 'Client Nonce:\n':
            parsing_clientnonce_line = 1
            cn = ""
        if 2 <= parsing_clientnonce_line <= 3:
            cn = cn + extract_data_from_line(line)

        if line == 'Master Secret:\n':
            parsing_mastersecret_line = 1
            ms = ""
        if 2 <= parsing_mastersecret_line <= 4:
            ms = ms + extract_data_from_line(line)

        if 5 == parsing_mastersecret_line:
            print('CLIENT_RANDOM', cn, ms)


if __name__ == '__main__':
    sys.exit(main())
```



获得了这个重要的pre-master参数后，就可以在wireshark中指定tls的pre-master-secret 文件的名字，于是可以解密https流量。



回归到要解决的问题，可以发现报出了这样的错误：

```
Frame 84: 538 bytes on wire (4304 bits), 538 bytes captured (4304 bits) on interface \Device\NPF_{1E884C5A-1654-4DF6-A7E8-01A37930B33B}, id 0
Ethernet II, Src: a2:39:20:00:08:00 (a2:39:20:00:08:00), Dst: 0f:00:08:00:00:00 (0f:00:08:00:00:00)
Internet Protocol Version 4, Src: xxx.yy.zzz.80, Dst: aa.bb.cc.dd
Transmission Control Protocol, Src Port: 443, Dst Port: 63636, Seq: 4343, Ack: 1042, Len: 484
Transport Layer Security
Hypertext Transfer Protocol
HTTP/1.1 400 Bad Request\r\n
[Expert Info (Chat/Sequence): HTTP/1.1 400 Bad Request\r\n]
[HTTP/1.1 400 Bad Request\r\n]
[Severity level: Chat]
[Group: Sequence]
Response Version: HTTP/1.1
Status Code: 400
[Status Code Description: Bad Request]
Response Phrase: Bad Request
Date: Thu, 19 Mar 2020 06:50:21 GMT\r\n
Server: Apache\r\n
Cache-Control: private\r\n
Content-Type: text/html; charset=utf-8\r\n
X-AspNet-Version: 4.0.30319\r\n
X-UA-Compatible: IE=edge\r\n
Vary: Accept-Encoding\r\n
Content-Encoding: gzip\r\n
Content-Length: 168\r\n
[Content length: 168]
Connection: close\r\n
\r\n
[HTTP response 1/1]
[Time since request: 0.325101000 seconds]
[Request in frame: 80]
[Request URI: https://xxx.yyy.com/mmm.services/api/oauth2/token]
Content-encoded entity body (gzip): 168 bytes -> 191 bytes
File Data: 191 bytes
Line-based text data: text/html (1 lines)
Error: SerializationException: Type definitions should start with a '{', expecting serialized type 'TokenRequest', got string starting with: client_secret=6e18dc44-1171-48b2-8380-66071231831e
```


通过错误可以猜测到和json的结构有关，于是找到代码中的相关部分进行更正，原来的请求就可以顺利执行了。





# 参考文献

https://zhuanlan.zhihu.com/p/44786952，

https://timothybasanov.com/2016/05/26/java-pre-master-secret.html



