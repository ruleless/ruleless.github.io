---
layout: post
title: "Radius身份认证协议"
description: "Radius身份认证协议"
category: radius
tags: []
---
{% include JB/setup %}

Radius常用的身份认证协议有：PAP/CHAP/MS-CHAP/MS-CHAP-V2。

当中，PAP是最简单的身份认证协议，用户密码以明文形式传输。
这对用户来讲是绝不可接受的，所以在这里不予讨论。

CHAP全称挑战握手认证协议，是"Challenge-Handshake Authentication Protocol"的简写。
CHAP/MS-CHAP/MS-CHAP-V2一脉相承，它们的认证过程基本类似：
先是服务端向客户端发起挑战，并附带一个16字节的挑战码；
然后客户端基于挑战码、用户名和用户输入的密码生成24字节的单向散列字串，再带上用户名一并发给服务端；
服务端根据用户名取得用户密码，然后通过与客户端相同的算法计算一个24字节的单向散列字串，
再将该字串与客户端发上来的散列字串作比较，如果相同，则认证成功，反之，则否。

需要注意的是，MS-CHAP-V2在上述认证过程之外还有一条反向认证过程，即客户端认证服务端，我们会在下面讨论到。

根据上面所说的认证过程，我们可以看到，用户密码是以不可逆的散列字串传输的，这保证了用户密码传输过程的安全性。
但仅提供单向认证过程的CHAP和MS-CHAP协议没有办法避免中间人攻击；而提供双向认证过程的MS-CHAP-V2协议则有效避免了这点。

为讨论方便，我们接下来用CHAPs指代CHAP/MS-CHAP/MS-CHAP-V2。

## PPP协议

在讨论CHAPs的协议细节之前，我们先简单介绍下PPP协议，因为CHAPs协议最初是用来作PPP链路上的身份认证的。

PPP下包含三种子协议：

  1. 用于封装通信数据的数据封包协议
  2. 链路控制协议(LCP: Link Control Protocol)，用于建立连接、协商身份认证协议、测试链路连接状况
  3. 网络控制协议

我们这里仅讨论LCP，因为我们关注的是身份认证这一块。

**当LCP协商的身份认证协议为PAP时，其格式如下：**

``` shell
| 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 |
|-----------------+-----------------+-----------------+-----------------|
| Type            | Length          | Authentication-Protocol           |
|-----------------+-----------------+-----------------+-----------------|
```

其中，Type=3，Length=4，Authentication-Protocol=0xC023。

**当LCP协商的身份认证协议为CHAPs时，其格式如下：**

``` shell
| 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 |
|-----------------+-----------------+-----------------+-----------------|
| Type            | Length          | Authentication-Protocol           |
|-----------------+-----------------+-----------------+-----------------|
| Algorithm       |
|-----------------+
```

其中，Type=3，Length=4，Authentication-Protocol=0xC223。
Algorithm指定了认证过程中的不同算法：

  1. 0x5: MD5
  2. 0x80: MS-CHAP
  3. 0x81: MS-CHAP-V2

## CHAP(MD5)

CHAP协议的格式如下：

``` shell
| 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 |
|-----------------+-----------------+-----------------+-----------------|
| Code            | Identifier      | Length                            |
|-----------------+-----------------+-----------------+-----------------|
| Data...
|-----------------
```

各协议字段的含义如下：

  + Code: 指定包类型，1 Challenge 2 Response 3 Success 4 Failure
  + Identifier: 包ID，用于匹配CHAP封包的请求和回复
  + Length: 整个CHAP封包的长度
  + Data: 根据不同的封包类型，此字段中的数据有不同的含义

Challenge和Response封包的格式如下：

``` shell
| 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 |
|-----------------+-----------------+-----------------+-----------------|
| Code            | Identifier      | Length                            |
|-----------------+-----------------+-----------------+-----------------|
| Value-Size      | Value...
|-----------------+-----------------+-----------------+-----------------
| Name...
|-----------------+-----------------+-----------------+-----------------
```

而Success和Failure封包的格式如下：

``` shell
| 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 |
|-----------------+-----------------+-----------------+-----------------|
| Code            | Identifier      | Length                            |
|-----------------+-----------------+-----------------+-----------------|
| Message...
|-----------------
```

CHAP的身份验证过程如下：

![](/images/radius/authentication/chap.png)

挑战握手协议的认证过程分三步：

  1. 服务端向客户端发送Challege发起挑战
  2. 客户端回应Response，计算公式是：Response=MD5(Challege, UserName, PlainPassword)，另外UserName会通过协议的Name字段带上去
  3. 服务端先根据传过来的UserName查找其明文密码，然后再以跟客户端同样的MD5算法计算得到一个Hash值，用该值跟Response比较，相同则认证成功，反之，则否

## MS-CHAP

MS-CHAP是微软提出的加强版CHAP。
根据上面的描述，CHAP认证过程必须依赖明文密码，服务端用户数据库必须要存储用户的明文密码。
而微软推出的Active Directory(AD域，一种LDAP实现)在保存用户账号信息的时候是不会存明文密码的，而代之以不可逆的散列码。
由于用户账号数据库不再存储明文密码，于是朴素的CHAP协议就不再适用了。

MS-CHAP的认证过程如下：

![](/images/radius/authentication/ms-chap.png)

这与CHAP的认证过程基本类似，唯一不同的是NT-Response的计算方式：
`NT-Response=NtChallengeResponse(AuthenticatorChallenge, PlainPassword)`。
NtChallengeResponse函数的实现伪码如下：

``` c++
NtChallengeResponse(
    IN 8-octet Challenge,
    IN 0-to-256-unicode-char Password,
    OUT 24-octet Response )
{
    NtPasswordHash( Password, giving PasswordHash );
    ChallengeResponse( Challenge, PasswordHash, giving Response );
}

NtPasswordHash(
    IN 0-to-256-unicode-char Password,
    OUT 16-octet PasswordHash)
{
    /*
     * Use the MD4 algorithm [5] to irreversibly hash Password
     * into PasswordHash. Only the password is hashed without
     * including any terminating 0.
     */
}

ChallengeResponse(
    IN 8-octet Challenge,
    IN 16-octet PasswordHash,
    OUT 24-octet Response)
{
    Set ZPasswordHash to PasswordHash zero-padded to 21 octets;
    DesEncrypt( Challenge,
                1st 7-octets of ZPasswordHash,
                giving 1st 8-octets of Response );
    DesEncrypt( Challenge,
                2nd 7-octets of ZPasswordHash,
                giving 2nd 8-octets of Response );
    DesEncrypt( Challenge,
                3rd 7-octets of ZPasswordHash,
                giving 3rd 8-octets of Response );
}
```

我们可以看到客户端的明文密码是经过了一次Hash运算才参与计算的，
这是为了获得跟服务端用户数据库一致的存储密码。

大多数时候，客户端不会直接跟Authenticator通信。
如手机PPTP或L2TP模块，它们会直接接入VPN，但大多数时候VPN服务端是不带AD域服务的。
所以在进行账号认证的时候，VPN需要先通过外部AD域服务器对用户进行身份认证。
其过程如下(我们以NAS指代VPN：NAS表示网络接入服务，在下面的场景下更具通用性)：

![](/images/radius/authentication/ms-chap-ad.png)

NAS自身产生了Challenge，但不能验证用户身份的合法性，当它收到客户端的Response后，
它将先前产生的Challenge和收到的Response以及客户端发上来的UserName转发给AD域服务器。
samba提供的winbind可以帮助完成此功能，事实上，现有的实现一般都采用此实现，如：FreeRadius。

## MS-CHAP-V2

MS-CHAP虽然解决了无明文密码的身份验证场景，但没有办法防止中间人攻击。
试想，如果在客户端与服务端的传输中间节点存在恶意服务器，它可以轻松冒充真正的服务器，
只需在挑战握手的第三步发送一个成功的认证封包，就能骗取客户端进行后续的数据通信。

MS-CHAP-V2针对此场景定义了双向认证过程，亦即，除了客户端向服务端认证自己，也需要服务端向客户端认证自己。
其过程如下：

![](/images/radius/authentication/ms-chap-v2.png)

相比MS-CHAP，我们可以看到上述过程在第二步，带上了客户端生成的Peer-Challenge；
而在第三步，服务端不仅要带上对客户端的认证结果，还需要带上auth_string。
auth_string是服务端通过某种算法生成的，其生成的全部要素客户端都有，
所以客户端也能根据与服务端相同的算法生成一个值来校验服务端发来的auth_string是否是对的。

上述过程为什么能防止中间人攻击呢？
防止中间人攻击的办法是校验只有客户端和服务端知道且不会在网络中传输的信息。
对MS-CHAP-V2而言，这个信息是什么呢？是用户的密码！

MS-CHAP-V2各变量的运算过程的伪码如下：

``` c++
NT-Response=GenerateNTResponse(AuthenticatorChallenge, PeerChallenge, UserName, Password);
auth_string=GenerateAuthenticatorResponse(Password, NT-Response, PeerChallenge, AuthenticatorChallenge,UserName);

GenerateNTResponse(
    IN 16-octet AuthenticatorChallenge,
    IN 16-octet PeerChallenge,
    IN 0-to-256-char UserName,
    IN 0-to-256-unicode-char Password,
    OUT 24-octet Response)
{
    ChallengeHash( PeerChallenge, AuthenticatorChallenge, UserName,
                   giving Challenge);
    NtPasswordHash( Password, giving PasswordHash );
    ChallengeResponse( Challenge, PasswordHash, giving Response );
}

ChallengeHash(
    IN 16-octet PeerChallenge,
    IN 16-octet AuthenticatorChallenge,
    IN 0-to-256-char UserName,
    OUT 8-octet Challenge)
{
    /*
     * SHAInit(), SHAUpdate() and SHAFinal() functions are an
     * implementation of Secure Hash Algorithm (SHA-1) [11]. These are
     * available in public domain or can be licensed from
     * RSA Data Security, Inc.
     */
    SHAInit(Context);
    SHAUpdate(Context, PeerChallenge, 16);
    SHAUpdate(Context, AuthenticatorChallenge, 16);
    /*
     * Only the user name (as presented by the peer and
     * excluding any prepended domain name)
     * is used as input to SHAUpdate().
     */
    SHAUpdate(Context, UserName, strlen(Username));
    SHAFinal(Context, Digest);
    memcpy(Challenge, Digest, 8);
}

NtPasswordHash(
    IN 0-to-256-unicode-char Password,
    OUT 16-octet PasswordHash )
{
    /*
     * Use the MD4 algorithm [5] to irreversibly hash Password
     * into PasswordHash. Only the password is hashed without
     * including any terminating 0.
     */
}

HashNtPasswordHash(
    IN 16-octet PasswordHash,
    OUT 16-octet PasswordHashHash )
{
    /*
     * Use the MD4 algorithm [5] to irreversibly hash
     * PasswordHash into PasswordHashHash.
     */
}

ChallengeResponse(
    IN 8-octet Challenge,
    IN 16-octet PasswordHash,
    OUT 24-octet Response )
{
    DesEncrypt( Challenge,
                1st 7-octets of ZPasswordHash,
                giving 1st 8-octets of Response );
    DesEncrypt( Challenge,
                2nd 7-octets of ZPasswordHash,
                giving 2nd 8-octets of Response );
    DesEncrypt( Challenge,
                3rd 7-octets of ZPasswordHash,
                giving 3rd 8-octets of Response );
}

GenerateAuthenticatorResponse(
    IN 0-to-256-unicode-char Password,
    IN 24-octet NT-Response,
    IN 16-octet PeerChallenge,
    IN 16-octet AuthenticatorChallenge,
    IN 0-to-256-char UserName,
    OUT 42-octet AuthenticatorResponse )
{
    /*
     * "Magic" constants used in response generation
     */
    Magic1[39] =
            {0x4D, 0x61, 0x67, 0x69, 0x63, 0x20, 0x73, 0x65, 0x72, 0x76,
             0x65, 0x72, 0x20, 0x74, 0x6F, 0x20, 0x63, 0x6C, 0x69, 0x65,
             0x6E, 0x74, 0x20, 0x73, 0x69, 0x67, 0x6E, 0x69, 0x6E, 0x67,
             0x20, 0x63, 0x6F, 0x6E, 0x73, 0x74, 0x61, 0x6E, 0x74};
    Magic2[41] =
            {0x50, 0x61, 0x64, 0x20, 0x74, 0x6F, 0x20, 0x6D, 0x61, 0x6B,
             0x65, 0x20, 0x69, 0x74, 0x20, 0x64, 0x6F, 0x20, 0x6D, 0x6F,
             0x72, 0x65, 0x20, 0x74, 0x68, 0x61, 0x6E, 0x20, 0x6F, 0x6E,
             0x65, 0x20, 0x69, 0x74, 0x65, 0x72, 0x61, 0x74, 0x69, 0x6F,
             0x6E};
    /*
     * Hash the password with MD4
     */
    NtPasswordHash( Password, giving PasswordHash );
    /*
     * Now hash the hash
     */
    HashNtPasswordHash( PasswordHash, giving PasswordHashHash);
    SHAInit(Context);
    SHAUpdate(Context, PasswordHashHash, 16);
    SHAUpdate(Context, NTResponse, 24);
    SHAUpdate(Context, Magic1, 39);
    SHAFinal(Context, Digest);
    ChallengeHash( PeerChallenge, AuthenticatorChallenge, UserName,
                   giving Challenge);
    SHAInit(Context);
    SHAUpdate(Context, Digest, 20);
    SHAUpdate(Context, Challenge, 8);
    SHAUpdate(Context, Magic2, 41);
    SHAFinal(Context, Digest);
    /*
     * Encode the value of ’Digest’ as "S=" followed by
     * 40 ASCII hexadecimal digits and return it in
     * AuthenticatorResponse.
     * For example,
     * "S=0123456789ABCDEF0123456789ABCDEF01234567"
     */
}
```
