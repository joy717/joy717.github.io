---
layout: post
author: joy717
---

**Here are 2 ways to create key and crt.**

## 1. Using golang tool:
https://golang.org/src/crypto/tls/generate_cert.go

generate ECC:

```
go run generate_cert.go --host joy717.com --ecdsa-curve P256 --duration 87600h
```

## 2. Using openssl

### ecc
#### create private key
`openssl ecparam -out server.key -name prime256v1 -genkey`

#### create csr
`openssl req -new -key server.key -out server.csr`

#### create crt
```
openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
```

### rsa

#### generate ca key/crt
```openssl req -new -x509 -newkey rsa:2048 -keyout ca.key -out ca.crt -days 36500```

#### generate server key
```openssl genrsa -out server.key 2048```

#### generate server csr
```openssl req -new -key server.key -out server.csr```

#### generate server crt
```openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 36500```






### 参考
* https://docs.azure.cn/zh-cn/articles/azure-operations-guide/application-gateway/aog-application-gateway-howto-create-self-signed-cert-via-openssl
