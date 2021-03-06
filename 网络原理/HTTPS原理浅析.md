### HTTPS原理浅析

#### 什么是HTTPS？
`HTTPS`并不是一个新的协议，它只是在`HTTP`协议上做了改进，解决了`HTTP`存在的一些问题：

1. 传输数据不进行任何数据加密，内容很容易被窃取。
2. 不验证对方的身份，因此有可能会遭遇三方的伪装。
3. 不校验报文内容的完整性，很可能报文内容已经遭遇了篡改。

`HTTPS`在原有`HTTP`的基础上，使用了`SSL/TLS`协议来解决上文讲到的问题，所以`HTTP`其实就是`HTTP`+`SSL/TLS`。

#### 什么是SSL/TLS？
`SSL`和`TLS`实际上是两个不同的协议，但是它们的作用都是相同的——保证数据传输过程中的安全性。

##### SSL
`SSL`(Secure Sockets Layer 安全套接字协议)是为网络通信提供安全及数据完整性的一种安全协议，由`Netscape`公司所研发，当前最新的版本为3.0。

##### TLS
TLS（Transport Layer Security，安全传输层)，TLS是建立在传输层TCP协议之上的协议，服务于应用层，它的前身是`SSL`。实际上，`TLS`是由`IETF`( 国际互联网工程任务组 The Internet Engineering Task Force)将`SSL`3.0标准化。现今最新版本的`TLS`协议和`SSL`协议几乎没有什么差异。

#### SSL/TLS原理
`SSL/TLS`协议是服务于应用层和传输层之间的，即应用层不再是直接和传输层通信，而是先和`SSL/TLS`通信，然后由`SSL/TLS`和传输层进行通信。

`HTTPS`实际上就是在通过三次握手建立`TCP`连接的时候，再进行一次`SSL/TLS`握手，完成`SSL`连接，建立安全的通信链路之后，就可以开始采用`HTTP`协议传输数据了，所以接下来要介绍的是`SSL/TLS`的原理，了解它是如何建立起安全的通信链路的。

在了解其实现原理之前，需要先掌握以下关于密码学的内容：

##### 密码学知识

**明文**：未经过加密的原始数据。

**密文**：经过加密算法处理之后的明文，可以保证数据的安全性。处理过后的数据看起来像是一串"乱码"，密文可以通过解密获取到明文。

**密钥**：密钥是一种参数，它是在明文转换为密文或将密文转换为明文的算法中输入的参数。简单点来说，它可以负责将明文转换成密文，也可以将密文还原成明文。密钥分两种：对称密钥和非对称密钥。

**对称密钥算法**：指加密处理和解密处理使用同一个密钥。**特点是算法公开、加密和解密速度快，适合于对大数据量进行加密**。常见的对称加密算法有DES、3DES、TDEA、Blowfish、RC5和IDEA。**缺点是一旦密钥泄露，那么密文就很容易被破解**。因此密钥的安全性管理相当重要。

**非对称加密**：与对称加密不同，这种加密方式使用两把不同的密钥，且二者成对出现。私钥自己保存，不能对外泄露。而公钥则是公开的，所有都可以获取获取公钥。明文可以通过公钥加密，私钥解密；也可以通过私钥加密，公钥解密。**相比起对称加密来说，非对称加密安全更加高**。**不过缺点也很明显，加密和解密花费时间长、速度慢，只适合对少量数据进行加密**。在非对称加密中使用的主要算法有：RSA、Elgamal、Rabin、D-H、ECC（椭圆曲线加密算法）等。

在`HTTPS`中，两种加密方式都有使用，非对称加密则是在验证服务端身份时使用，而对称加密是在身份验证完成后，开始真正的数据传输时使用。

下面开始讲解`SSL/TLS`的原理，看看它是如何解决`HTTP`存在的问题。

##### 身份验证
上面说到了一个验证服务器身份，前面提到，`HTTP`请求无法验证身份，也就是说，客户端不知道请求的是不是一台合法的服务器。

而`SSL/TLS`提供了一种叫做**数字证书**的身份验证机制，使得客户端在通信之前能够验证服务端的身份，从而防止第三方冒充。

数字证书就是互联网通讯中标志通讯各方身份信息的一串数字，提供了一种在互联网上验证通信实体身份的方式。简单来说，数字证书可以当做个人或者企业在互联网的身份证。

举个例子：你要去银行取钱，你要如何向银行证明你就是你，而什么可以证明你就是你呢？答案就是你的身份证，通过身份证，银行就能够确认你就是这张银行卡的持有者。

数字证书的来源一般来自于三方的权威机构，叫做`CA`（证书管理机构）所颁发。申请`CA`的数字证书需要经过非常严格的审核，因此拥有`CA`颁发的数字证书的服务器，都可以视为合法的服务器，就像我们的身份证，它有公安局颁布的，因此是绝对可靠的。

数字证书还可以自己制作自己颁发，不过由于不是`CA`颁发，浏览器会将其视为不安全的并弹窗警告用户，下文会讲解原因。

数字证书一般会有以下的内容：

1. 证书所有者（证书的接收者）
2. 证书的发行机构
3. 证书所有者的公钥内容
4. 过期时间
5. 机构的数字签名(用于验证)
    ...

其中1-4属于明文，而5则属于密文，下文讲客户端验证证书的时候会讲。
##### 证明证书的合法性和完整性

进行身份验证的时候，服务器需要将证书发送给客户端，但是在传送的过程中，万一证书被伪造，被篡改该怎么办呢？因此客户端收到服务器发送过来的证书之后，还需要验证证书的合法性和完整性。在`SSL/TLS`中，使用了**数字摘要**和**数字签名**用来让客户端验证接收到的证书。

先讲讲数字摘要，不过在了解数字摘要之前，需要了解一种算法叫做**HASH算法**，这种算法可以**将任意长度的输入，转化成为固定长度的输出**，这个输出又称作散列值。并且，这种算法输入值和输出值是一对一的关系，也就是说，如果输入值有所改动，那么输出值必定不一样，而且该过程是单向的，只能输入—>输出，而不能从输出—>输入。

**数字摘要**是由数字证书的明文内容部分进行`HASH`处理，得到的一个编码串。当客户端得到数字证书时，利用相同的`HASH`算法对明文部分进行处理，同样也会得到一个摘要，然后和证书中的数字摘要对比是否完全相同，如果完全相同就表示内容没有被篡改过。

不过仅仅如此还不够，当前只是验证了内容没有被篡改，不过还没有证明证书的合法性。因此，**数字签名**的作用显示出来了，数字签名就像平时签订协议或者下发通知时的公章或者签名，用来表示文件的有效性，也表示了签发者对文件内容的肯定。

那究竟是如何进行数字签名的呢？这里就要用到非对称加密算法了，`CA`在给服务器颁发数字证书的时候，先利用`HASH`算法将明文内容生成数字摘要，然后再利用它自己的私钥，**对数字摘要进行加密，得到的加密串就是数字签名**，数字签名作为数字证书的一部分一起发送给客户端。

而公钥在哪里呢？上文说了，非对称加密的算法的公钥是公开的，而`CA`的公钥，就在称为**根证书**的东西中，这一类证书一般安装在我们的浏览器上，浏览器一般会预装一些`CA`机构的根证书，当客户端接收到数字证书和数字签名的时候，就会先通过证书的上的颁发机构，寻找是否存在其根证书，如果找不到则表示该证书并不是由权威的`CA`机构所颁发，此时客户端就会警告该链接不安全。

这里梳理一下客户端验证数字证书的过程：
1. 客户端接收到数字证书和数字签名之后，寻找本地是否存在对应的公钥进行解密，找不到则警告该连接不安全。
2. 客户端先采用相同的`HASH`算法对数字证书的明文进行处理，得到一个摘要`H1`
3. 使用本地的公钥，对数字签名进行解密，得到摘要`H2`
4. 对比`H1`和`H2`,如果一模一样则表示内容没被篡改。

由于数字签名必须使用私钥，因此具有**不可抵赖性、不可伪造性**，因为私钥并不公开，只要客户端能成功利用公钥解密，那么就能证明是该私钥所有者发送的数据。

#### HTTPS连接建立过程

了解完上文内容之后，接下来，就来了解一下整个`HTTPS`建立连接的过程，整个`HTTPS`建立过程一共使用了三对密钥：分别是`CA`的非对称加密密钥、服务端的非对称加密密钥、客户端的对称机密密钥。

服务端通过向`CA`机构提交指定资料（服务端的公钥以及一些其他申请信息、比如使用的HASH算法等）发送给`CA`，如果申请通过的话，就会经历以下步骤：

1. `CA`首先将服务端提交的信息加上一些自身的内容(明文)，然后利用`HASH`算法进行处理，得到数字摘要。
2. 然后再利用自身的私钥对摘要进行签名。
3. 将明文内容和数字签名组装成数字证书，然后发送给服务端。

当客户端向服务端发送请求时，如果发现是https连接，就会向服务端询问证书，此时服务端就会将`CA`颁布的证书发送给客户端，客户端接收到证书后，开始以下验证过程：

1. 利用相同的`HASH`算法将明文处理成摘要`H1`。
2. 获取本地根证书中的公钥之后，利用公钥对服务端发送过来的签名进行解密，获取到摘要`H2`
3. 对比`H1`和`H2`，如果一模一样，则验证成功，表示服务端是真实可靠的。

客户端验证完证书之后，获取服务端提供的公钥，然后生一个对称加密的密钥，通过服务端提供的公钥进行加密，再发送给服务端。

服务端接收到之后，利用自己的私钥进行解密，获得了客户端的对称加密密钥，自此，数据传输正式开始，传输过程采用的是客户端提供的对称加密密钥。

