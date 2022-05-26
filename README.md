# WSO2-deployment

## Purpose
This repos purpose is to install a wso2am 4.1.0 single node installation on my private Server running a Kubernetes server.
This will be done using helm chart and Jenkins pipelines to automate this procces.
Sience this installation is optimizaed for my deployment i will clean up all unesarcy configuration in the deployment to keep the instllation and configuration down for my self.
I will still enable full security hardening as recomended by WSO2 that is not present in the default configuration.

## Installation

### Database
Install postgress localy on you machine and follow this guide
https://apim.docs.wso2.com/en/latest/install-and-setup/setup/setting-up-databases/changing-default-databases/changing-to-postgresql/

### Certification generation
Here we generate the internal Certificate used for encryption and internal comunication in the Cluster.
We are not genertaing the external certificate that is handeld by our global CA, example https://letsencrypt.org/

**TLS-certificate**

Get the Cluster IP
kubectl get services


Create a file named csr.conf that will contain the configuration for the TLS certificate, replace all 5 tags <tags>
csr.conf

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn
 
[ dn ]
C = SE
ST = Sweden
L = Stockholm
O = MyHome
OU = office
CN = wso2am-am-service
 
[ req_ext ]
subjectAltName = @alt_names
 
[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
DNS.6 = wso2am-am-service
DNS.9 = wso2am-am-gateway-service
DNS.11 = <gw-dns>
DNS.12 = <webbsub-dns>
IP.1 = <Cluster IP>
IP.2 = <server IP>

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

Generate the CA key
> openssl genrsa -out ca.key 2048


Create the CA certificate from the Key, replace the <Cluster IP> with the cluster IP 
> openssl req -x509 -new -nodes -key ca.key -subj "/CN={Cluster IP}" -days 20000 -out ca.crt

> openssl genrsa -out server.key 2048

> openssl req -new -key server.key -out server.csr -config csr.conf

> openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 20000 -extensions v3_ext -extfile csr.conf

> openssl pkcs12 -export -in server.crt -inkey server.key -name servertls -out servertls.p12

> keytool -importkeystore -srckeystore servertls.p12 -srcstoretype pkcs12 -srcalias servertls -destkeystore tls-keystore.jks -deststoretype JKS  -destalias servertls


 
#delete Cert
> keytool -delete -noprompt -alias servercert  -keystore client-truststore.jks -storepass wso2carbon

> keytool -importcert -file server.crt -keystore client-truststore.jks -alias "servercert" -storepass wso2carbon

Encryption Key
This describe how to create the WSO2 Encryption key use to encrypt all passwords and configuration
Start by creating a file named encryption-request.cfg that contains the encryption configuration
encryption-request.cfg
```
[req]
x509_extensions = v3_req
prompt = no
distinguished_name = dn
 
[ dn ]
C = SE
ST = Sweden
L = Stockholm
O = MyHome
OU = office
CN = wso2am-am-service
 
[v3_req]
keyUsage = keyEncipherment, dataEncipherment, digitalSignature
extendedKeyUsage = serverAuth, clientAuth
```
use Openssl to create the encryption key
> openssl req -x509 -nodes -days 10000 -newkey rsa:4096 -keyout encryption.key -out encryption.crt -config encryption-request.cfg -extensions 'v3_req'

Move the encryption key and certificate into a p12 container
Note: Use the same password for key and keystore
> openssl pkcs12 -export -in encryption.crt -inkey encryption.key -name encryptkey -out encryptkey.p12

Convert the p12 container to the Java keystore in jks format
Note: Use the same password for key and keystore
> keytool -importkeystore -srckeystore encryptkey.p12 -srcstoretype pkcs12 -srcalias encryptkey -destkeystore internal-keystore.jks -deststoretype JKS -destalias encryptkey

### Secrets

