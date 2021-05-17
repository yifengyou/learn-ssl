<!-- MDTOC maxdepth:6 firsth1:1 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [go单向验证](#go单向验证)   
   - [实验环境](#实验环境)   
   - [创建CA及服务端、客户端证书](#创建ca及服务端、客户端证书)   
   - [服务端代码](#服务端代码)   
   - [客户端代码](#客户端代码)   
   - [测试](#测试)   

<!-- /MDTOC -->
# go单向验证

## 实验环境

1. VMware 15
2. Fedora 34 虚拟机两台

```
192.168.30.150  tls-node1.test.com
192.168.30.151  tls-node2.test.com
```



## 创建CA及服务端、客户端证书

1. 创建我们自己CA(Certificate authority)的私钥：

```
openssl genrsa -out ca.key 2048
```

2. 创建我们自己CA(Certificate authority)的CSR(Certificate Signing Request)，并且用自己的私钥自签署之，得到CA的身份证：

```
openssl req -x509 -new -nodes -key ca.key -days 10000 -out ca.crt -subj "/CN=we-as-ca"
```

3. 创建server的私钥，CSR(Certificate Signing Request)，并且用**CA的私钥ca.key**自签署server的身份证CRT(Certificate)：

```
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=tls-node1.test.com"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```

其中CN(Common Name)必须与服务器端域名保持一致


## 服务端代码

```
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
)

func main() {
	s := &http.Server{
		Addr: ":443",
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			fmt.Fprintf(w, "Hello World!\n")
		}),
		TLSConfig: &tls.Config{
			ClientCAs:  loadCA("ca.crt")
		},
	}

	e := s.ListenAndServeTLS("server.crt", "server.key")
	if e != nil {
		log.Fatal("ListenAndServeTLS: ", e)
	}
}

func loadCA(caFile string) *x509.CertPool {
	pool := x509.NewCertPool()

	if ca, e := ioutil.ReadFile(caFile); e != nil {
		log.Fatal("ReadFile: ", e)
	} else {
		pool.AppendCertsFromPEM(ca)
	}
	return pool
}
```

## 客户端代码

```
package main

import (
	"crypto/tls"
	"crypto/x509"
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"os"
)

func main() {
	client := &http.Client{
		Transport: &http.Transport{
			TLSClientConfig: &tls.Config{
				RootCAs:      loadCA("ca.crt"),
			},
		}}

	resp, e := client.Get("https://tls-node1.test.com")
	if e != nil {
		log.Fatal("http.Client.Get: ", e)
	}
	defer resp.Body.Close()
	io.Copy(os.Stdout, resp.Body)
}

func loadCA(caFile string) *x509.CertPool {
	pool := x509.NewCertPool()

	if ca, e := ioutil.ReadFile(caFile); e != nil {
		log.Fatal("ReadFile: ", e)
	} else {
		pool.AppendCertsFromPEM(ca)
	}
	return pool
}
```

## 测试

![20210517_164651_36](image/20210517_164651_36.png)

报错，这是为何？

```
2021/05/17 04:46:45 http.Client.Get: Get "https://tls-node1.test.com": x509: certificate relies on legacy Common Name field, use SANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0
exit status 1
```

1. golang 1.15+版本更改策略
2. 证书，并没有开启SAN扩展（默认是没有开启SAN扩展）所生成的，所以在导致客户端和服务端无法建立连接

![20210517_172001_58](image/20210517_172001_58.png)






































---
