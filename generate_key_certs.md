**Here are 2 ways to create key and crt.**

## 1. Using golang tool:
https://golang.org/src/crypto/tls/generate_cert.go

generate ECC:

```
go run generate_cert.go --host joy717.com --ecdsa-curve P256 --duration 87600h
```

## 2. Using openssl

### create private key
`openssl ecparam -out server.key -name prime256v1 -genkey`

### create csr
`openssl req -new -key server.key -out server.csr`

### create crt
```
openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
```






### 参考
* https://docs.azure.cn/zh-cn/articles/azure-operations-guide/application-gateway/aog-application-gateway-howto-create-self-signed-cert-via-openssl
