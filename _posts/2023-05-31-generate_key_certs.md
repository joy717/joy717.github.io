---
title: 生成自签名证书
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



### online command
```
openssl req -x509 -newkey rsa:2048 -sha256 -days 3650 -nodes \
  -keyout tls.key -out tls.crt -subj '/CN=vintest.com' \
  -addext 'subjectAltName=DNS:vintest.com,DNS:www.vintest.com'
```


### common style
```
openssl req -x509 -new -nodes -sha512 -days 3650 \
    -subj "/C=CN/ST=XiaMen/L=XiaMen/O=example/OU=Personal/CN=vintest.com" \
    -key ca.key \
    -out ca.crt
    
    
    
    openssl genrsa -out server.key 4096
    
    
    openssl req -sha512 -new \
    -subj "/C=CN/ST=XiaMen/L=XiaMen/O=example/OU=Personal/CN=vintest.com" \
    -key server.key \
    -out server.csr 
    
    
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth 
subjectAltName = @alt_names

[alt_names]
IP.1=172.16.12.172
DNS.1=vintest.com
EOF


openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in server.csr \
    -out server.crt
```


## 校验

```
# 查看KEY信息

> openssl rsa -noout -text -in myserver.key

# 查看CSR信息

> openssl req -noout -text -in myserver.csr

# 查看证书信息

> openssl x509 -noout -text -in ca.crt
```




### 参考
* https://docs.azure.cn/zh-cn/articles/azure-operations-guide/application-gateway/aog-application-gateway-howto-create-self-signed-cert-via-openssl
