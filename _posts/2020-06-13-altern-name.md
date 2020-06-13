---
title: "证书签发与 SubjectAltName 扩展项"
date: 2020-06-13  16:29:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "openssl"
    - "数字证书"
---

## subjectAltName 的作用

subjectAltName 在 [RFC 5280 4.2.1.6.](https://www.ietf.org/rfc/rfc5280.txt) 中提供了详细的说明，subjectAltName 是 X.509 version 3 的一个扩展项，该扩展项用于标记和界定证书持有者的身份。

在 X.509 格式的证书中，一般使用 `Issuer` 项标记证书的颁发者信息，该项必须是一个非空的 Distinguished Name 名称。除此之外还可以使用扩展项 `issuerAltName` 来标记颁发者的其他名称，这是一个非关键的扩展项。

对于证书持有者，一般使用 `Subject` 项标记，并使用 `subjectAltName` 扩展项提供更详细的持有者身份信息。 `subjectAltName`  全称为 Subject Alternative Name，缩写为 SAN。它可以包括一个或者多个的电子邮件地址，域名，IP地址和 URI 等，详细定义如下：

```
   SubjectAltName ::= GeneralNames
   GeneralNames ::= SEQUENCE SIZE (1..MAX) OF GeneralName

   GeneralName ::= CHOICE {
        otherName                       [0]     OtherName,
        rfc822Name                      [1]     IA5String,
        dNSName                         [2]     IA5String,
        x400Address                     [3]     ORAddress,
        directoryName                   [4]     Name,
        ediPartyName                    [5]     EDIPartyName,
        uniformResourceIdentifier       [6]     IA5String,
        iPAddress                       [7]     OCTET STRING,
        registeredID                    [8]     OBJECT IDENTIFIER 
    }
```

当颁发的证书不存在 `Subject` 项的时候，证书必须包含扩展项 `subjectAltName`，并且标记为关键（critical）的。当颁发的证书存在 `Subject` 项的时候，必须将扩展项 `subjectAltName` 标记为非关键（no-critical）的。注意：用于颁发证书的 CA 证书是必须包含 `Subject` 项的。

根据 [RFC 6125](https://www.ietf.org/rfc/rfc6125.txt) 中的规定，当一个网站使用证书标记自己的身份时，如果证书中包含 subjectAltName，在识别证书持有者时会忽略 Subject 子项，而是通过 subjectAltName 来识别证书持有者。在早期颁发的证书中一般通过 Subject 的 CommonName 来识别持有者的身份，不包含 subjectAltName 扩展项。这会导致最新版本的浏览器Chrome、Firefox 等在通过 HTTPS 访问 web 网站时，触发 NET::ERR_CERT_COMMON_NAME_INVALID 错误。

## Java TLS 中的检查过程

Java 在 TLS 建立的过程中，默认通过 `sun.security.util.HostnameChecker` 进行证书持有者身份检查。检查流程如下：

1. 从 SSLSession 中获取到服务端的地址信息 `SSLSession#getPeerHost()`，这个地址实际上是 Socket 建立连接的时候指定的 IP 地址或域名地址。

2. 判断该地址是 IP 地址还是域名地址。

    ```java
        public void match(String host, X509Certificate serverCert) throws CertificateException {
            if (isIpAddress(host)) {
                matchIP(host, serverCert);
            } else {
                this.matchDNS(host, serverCert);
            }
        }
    ```

3. 如果是 IP 地址，则在 subjectAltName 中寻找 IP 进行匹配，根据  [RFC 5280 4.2.1.6.](https://www.ietf.org/rfc/rfc5280.txt)  中对 GeneralName 的定义 IP 为类型 7。

    ```java
    private static void matchIP(String host, X509Certificate certificate) throws CertificateException {
        Collection san = certificate.getSubjectAlternativeNames();
        if (san == null) {
            throw new CertificateException("No subject alternative names present");
        } else {
            Iterator generalNames = san.iterator();
            while(generalNames.hasNext()) {
                List var4 = (List)generalNames.next();
                if ((Integer)var4.get(0) == 7) {
                    String generalName = (String)var4.get(1);
                    if (host.equalsIgnoreCase(generalName)) {
                        return;
                    }
                    try {
                        if (InetAddress.getByName(host).equals(InetAddress.getByName(generalName))) {
                            return;
                        }
                    } catch (UnknownHostException var7) {
                    } catch (SecurityException var8) {
                    }
                }
            }
            throw new CertificateException("No subject alternative names matching IP address " + host + " found");
        }
    }
    ```

4. 如果是域名地址，则在 subjectAltName 中寻找域名进行匹配，根据  [RFC 5280 4.2.1.6.](https://www.ietf.org/rfc/rfc5280.txt)  中对 GeneralName 的定义域名为类型 2。注意：当使用域名的时候，除了检查 subjectAltName 中是否存在匹配的域名之外，还会检查 Subject 中的 commonName 是否和域名匹配。

    ```java
    private void matchDNS(String host, X509Certificate certificate) throws CertificateException {
        try {
            new SNIHostName(host);
        } catch (IllegalArgumentException e) {
            throw new CertificateException("Illegal given domain name: " + host, e);
        }
    
        Collection san = certificate.getSubjectAlternativeNames();
        if (san != null) {
            boolean unMatched = false;
            Iterator generalNames = san.iterator();
    
            while(generalNames.hasNext()) {
                List generalName = (List)generalNames.next();
                if ((Integer)generalName.get(0) == 2) {
                    unMatched = true;
                    String var7 = (String)generalName.get(1);
                    if (this.isMatched(host, var7)) {
                        return;
                    }
                }
            }
    
            if (unMatched) {
                throw new CertificateException("No subject alternative DNS name matching " + host + " found.");
            }
        }
    
        X500Name DN = getSubjectX500Name(certificate);
        DerValue commonName = DN.findMostSpecificAttribute(X500Name.commonName_oid);
        if (var11 != null) {
            try {
                if (this.isMatched(host, commonName.getAsString())) {
                    return;
                }
            } catch (IOException e) {
            }
        }
    
        String err = "No name matching " + host + " found";
        throw new CertificateException(err);
    }
    ```

## 配置和脚本

自签发 CA 证书配置文件` ca-openssl.cnf`：

```ini
[req]
distinguished_name  = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]
countryName           = Country Name (2 letter code)
countryName_default = CN
stateOrProvinceName   = State or Province Name (full name)
stateOrProvinceName_default = Some-State
organizationName          = Organization Name (eg, company)
organizationName_default = Internet Widgits Pty Ltd
commonName            = Common Name (eg, YOUR name)
commonName_default = testca

[v3_req]
basicConstraints = CA:true
keyUsage = critical, keyCertSign
```

服务端证书配置文件 `server-openssl.cnf`：

```ini
[req]
distinguished_name  = req_distinguished_name
req_extensions     = v3_req

[req_distinguished_name]
countryName           = Country Name (2 letter code)
countryName_default   = CN
stateOrProvinceName   = State or Province Name (full name)
stateOrProvinceName_default = Beijing
localityName          = Locality Name (eg, city)
localityName_default  = Beijing
organizationName          = Organization Name (eg, company)
organizationName_default  = Example, Co.
commonName            = Common Name (eg, YOUR name)
commonName_max        = 64

####################################################################
[ ca ]
default_ca	= CA_default		# The default ca section

####################################################################
[ CA_default ]

dir		= . # Where everything is kept
certs		= $dir # Where the issued certs are kept
crl_dir		= $dir		# Where the issued crl are kept
database	= $dir/index.txt	# database index file.
#unique_subject	= no			# Set to 'no' to allow creation of
					# several ctificates with same subject.
new_certs_dir	= $dir		# default place for new certs.

certificate	= $dir/ca.pem 	# The CA certificate
serial		= $dir/serial 		# The current serial number
crlnumber	= $dir/crlnumber	# the current crl number
					# must be commented out to leave a V1 CRL
crl		= $dir/crl.pem 		# The current CRL
private_key	= $dir/private/cakey.pem# The private key
RANDFILE	= $dir/private/.rand	# private random number file

x509_extensions	= usr_cert		# The extentions to add to the cert

# Comment out the following two lines for the "traditional"
# (and highly broken) format.
name_opt 	= ca_default		# Subject Name options
cert_opt 	= ca_default		# Certificate field options

# Extension copying option: use with caution.
# copy_extensions = copy

# Extensions to add to a CRL. Note: Netscape communicator chokes on V2 CRLs
# so this is commented out by default to leave a V1 CRL.
# crlnumber must also be commented out to leave a V1 CRL.
# crl_extensions	= crl_ext

default_days	= 365			# how long to certify for
default_crl_days= 30			# how long before next CRL
default_md	= default		# use public key default MD
preserve	= no			# keep passed DN ordering

# A few difference way of specifying how similar the request should look
# For type CA, the listed attributes must be the same, and the optional
# and supplied fields are just that :-)
policy		= policy_anything
[ policy_anything ]
countryName		= optional
stateOrProvinceName	= optional
localityName		= optional
organizationName	= optional
organizationalUnitName	= optional
commonName		= supplied
emailAddress		= optional

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
IP.1 = 10.0.2.70

```

这里的 `[alt_names]` 域中即为我们需要指定的 `subjectAltName`，可以配置多个 IP，DNS 或其他值。

下面是一个用于证书签发，证书文件转换的脚本 `certificate-gen.sh`。使用该脚本前需要确认有 Java 的 keytool 命令和 openssl 命令。该脚本会生成一张 CA 自签发证书，然后使用该证书签发一张客户端证书和服务端证书。该脚本运行时会要求输入一个服务端 IP 的地址，这个地址会替换  `[alt_names]`  域的 `IP.1` 的值。下面是脚本内容：

```bash
#! /bin/bash
echo "INFO: 清理环境"
rm *.rsa
rm *.jks
rm *.p12
rm *.key
rm *.csr
rm *.srl

echo "INFO: 生成自签发证书"
openssl req -x509 -new -newkey rsa:2048 -nodes -keyout ca.key -out ca.pem -config ca-openssl.cnf -days 3650 -extensions v3_req

echo "INFO: 将 ca.pem 转换为 ca.jks, KeyStore 密码为 123456"
keytool -importcert -trustcacerts -file ca.pem -keystore ca.jks -storepass 123456

echo "INFO: 签发客户端证书"
openssl genrsa -out client.key.rsa 2048
openssl pkcs8 -topk8 -in client.key.rsa -out client.key -nocrypt
openssl req -new -key client.key -out client.csr
openssl x509 -req -CA ca.pem -CAkey ca.key -CAcreateserial -in client.csr -out client.pem -days 3650

echo "INFO: 将私钥和对应的证书链合成 PKCS#12 格式，KeyStore 密码和私钥密码均为 123456"
openssl pkcs12 -export -CAfile ca.pem -in client.pem  -inkey client.key -out client.p12 -passout pass:123456

echo "INFO: 签发服务端证书"
echo "INFO: 填写主机名"
read -p "请输入服务器域名或者主机名：" server
echo "INFO: set alt_names $server"
old_server=$(grep "IP.1 = " server-openssl.cnf|awk -F " " '{print $3}')
echo "INFO: 将 alt_names 从 $old_server 修改为 $server"
sed -i "s/$old_server/$server/g" server-openssl.cnf
openssl genrsa -out server.key.rsa 2048
openssl pkcs8 -topk8 -in server.key.rsa -out server.key -nocrypt
openssl req -new -key server.key -out server.csr -config server-openssl.cnf
openssl x509 -req -CA ca.pem -CAkey ca.key -CAcreateserial -in server.csr -out server.pem -extensions v3_req -extfile server-openssl.cnf -days 3650

echo "INFO: 将私钥和对应的证书链合成 PKCS#12 格式，KeyStore 密码和私钥密码均为 123456"
openssl pkcs12 -export -CAfile ca.pem -in server.pem  -inkey server.key -out server.p12 -passout pass:123456

echo "INFO：清理无用的文件"
rm *.rsa
rm *.csr
rm ca.srl
```

运行脚本前的目录结构：

```
certificates
├── ca-openssl.cnf
├── certificate-gen.sh
└── server-openssl.cnf
```

运行脚本后的目录结构：

```
certificates
├── ca-openssl.cnf
├── ca.jks
├── ca.key
├── ca.pem
├── certificate-gen.sh
├── client.key
├── client.p12
├── client.pem
├── server-openssl.cnf
├── server.key
├── server.p12
└── server.pem

0 directories, 12 files
```





