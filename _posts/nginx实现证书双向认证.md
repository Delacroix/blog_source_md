---
title: nginx实现证书双向认证
date: 2017-11-30 11:12:30
tags:
- nginx
- 安全
- CA证书
---

实现的功能：外网访问nginx有证书才能打开页面，无证书打不开页面
 
 
**一、在192.168.168.168安装nginx**

```
apt-get install nginx
```

 
 
**二、配置生成ca的参数**

```
mkdir -pv /etc/nginx/ca
cd /etc/nginx/ca
mkdir -pv mkdir newcerts private conf server
cd conf
 
vim openssl.conf
[ ca ] 
default_ca      = foo                   # The default ca section 
    
[ foo ] 
dir            = /etc/nginx/ca         # top dir 
database       = /etc/nginx/ca/index.txt          # index file. 
new_certs_dir  = /etc/nginx/ca/newcerts           # new certs dir 
    
certificate    = /etc/nginx/ca/private/ca.crt         # The CA cert 
serial         = /etc/nginx/ca/serial             # serial no file 
private_key    = /etc/nginx/ca/private/ca.key  # CA private key 
RANDFILE       =/etc/nginx/ca/private/.rand      # random number file 
    
default_days   = 365                     # how long to certify for 
default_crl_days= 30                     # how long before next CRL 
default_md     = md5                     # message digest method to use 
unique_subject = no                      # Set to 'no' to allow creation of 
 
policy         = policy_any
    
[ policy_any ] 
countryName = match 
stateOrProvinceName = match 
organizationName = match 
organizationalUnitName = match 
localityName            = optional 
commonName              = supplied 
emailAddress            = optional 
```

 
 
**三、生成根证书**

```
cd ..
vim new_ca.sh
#!/bin/sh 
# Generate the key. 
openssl genrsa -out private/ca.key
# Generate a certificate request. 
openssl req -new -key private/ca.key -out private/ca.csr
# Self signing key is bad... this could work with a third party signed key... registeryfly has them on for $16 but I'm too cheap lazy to get one on a lark. 
# I'm also not 100% sure if any old certificate will work or if you have to buy a special one that you can sign with. I could investigate further but since this 
# service will never see the light of an unencrypted Internet see the cheap and lazy remark. 
# So self sign our root key. 
openssl x509 -req -days 365 -in private/ca.csr -signkey private/ca.key -out private/ca.crt
# Setup the first serial number for our keys... can be any 4 digit hex string... not sure if there are broader bounds but everything I've seen uses 4 digits. 
echo FACE > serial
# Create the CA's key database. 
touch index.txt
# Create a Certificate Revocation list for removing 'user certificates.' 
openssl ca -gencrl -out /etc/nginx/ca/private/ca.crl -crldays 7 -config "/etc/nginx/ca/conf/openssl.conf"
 
chmod 777 new_ca.sh
```

执行生成根证书，记住输入的内容，之后生成服务器证书和客户端证书输入内容要相同

```
./new_ca.sh
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:cq
Locality Name (eg, city) []:cq
Organization Name (eg, company) [Internet Widgits Pty Ltd]:iot
Organizational Unit Name (eg, section) []:ops
Common Name (e.g. server FQDN or YOUR name) []:www.xxx.com
Email Address []:
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:!xxx
An optional company name []:xxx
```

 
 
**四、生成服务器证书**

```
vim new_server.sh
# Create us a key. Don't bother putting a password on it since you will need it to start apache. If you have a better work around I'd love to hear it. 
openssl genrsa -out server/server.key
# Take our key and create a Certificate Signing Request for it. 
openssl req -new -key server/server.key -out server/server.csr
# Sign this bastard key with our bastard CA key. 
openssl ca -in server/server.csr -cert private/ca.crt -keyfile private/ca.key -out server/server.crt -config "/etc/nginx/ca/conf/openssl.conf"
 
 

chmod 777 new_server.sh
./new_server.sh
```

执行中输入内容要和根证书相同
 
 
**五、修改nginx配置文件**

```
vim /etc/nginx/nginx.conf
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  logstash_json  '{ "@timestamp": "$time_local", '
                               '"@fields": { '
                               '"remote_addr": "$remote_addr", '
                               '"remote_user": "$remote_user", '
                               '"body_bytes_sent": "$body_bytes_sent", '
                               '"request_time": "$request_time", '
                               '"status": "$status", '
                               '"request": "$request", '
                               '"request_method": "$request_method", '
                               '"http_referrer": "$http_referer", '
                               '"body_bytes_sent":"$body_bytes_sent", '
                               '"http_x_forwarded_for": "$http_x_forwarded_for", '
                               '"http_user_agent": "$http_user_agent" } }';
    access_log     off;
    sendfile        on;
    keepalive_timeout  65;
    server_tokens    off;
 
    upstream nginx {
        server 192.168.168.168:80 weight=1 max_fails=2 fail_timeout=1;
        }
 
    server {
        listen       8989;
        access_log  /var/log/nginx/accesse.log  logstash_json;
        server_name  www.xxx.com;
 
        ssi on;
        ssi_silent_errors on;
        ssi_types text/shtml; 
   
        ssl                  on; 
        ssl_certificate      /etc/nginx/ca/server/server.crt; 
        ssl_certificate_key  /etc/nginx/ca/server/server.key; 
        ssl_client_certificate /etc/nginx/ca/private/ca.crt; 
   
        ssl_session_timeout  5m; 
        ssl_verify_client on;
   
        ssl_protocols  SSLv2 SSLv3 TLSv1; 
        ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP; 
        ssl_prefer_server_ciphers   on;
 
        location / {
            proxy_pass   http://nginx/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            }
        }
 
    include /etc/nginx/conf.d/*.conf;
}
```

 
 
 
**六、生成客户端证书并访问**

```
vim new_user.sh
#!/bin/sh 
# The base of where our SSL stuff lives. 
base="/etc/nginx/ca"
# Were we would like to store keys... in this case we take the username given to us and store everything there. 
mkdir -p $base/users/ 
   
# Let's create us a key for this user... yeah not sure why people want to use DES3 but at least let's make us a nice big key. 
openssl genrsa -des3 -out $base/users/client.key 1024 
# Create a Certificate Signing Request for said key. 
openssl req -new -key $base/users/client.key -out $base/users/client.csr 
# Sign the key with our CA's key and cert and create the user's certificate out of it. 
openssl ca -in $base/users/client.csr -cert $base/private/ca.crt -keyfile $base/private/ca.key -out $base/users/client.crt -config "/etc/nginx/ca/conf/openssl.conf"
   
# This is the tricky bit... convert the certificate into a form that most browsers will understand PKCS12 to be specific. 
# The export password is the password used for the browser to extract the bits it needs and insert the key into the user's keychain. 
# Take the same precaution with the export password that would take with any other password based authentication scheme. 
openssl pkcs12 -export -clcerts -in $base/users/client.crt -inkey $base/users/client.key -out $base/users/client.p12
 
chmod 777 new_user.sh
./new_user.sh
```

执行中输入内容要和根证书相同
 
将生成的客户端证书导入浏览器，注意导入时选择自动导入，密码：xxx

```
sz /etc/nginx/ca/users/client.p12
```

 
修改/etc/hosts加入

```
183.183.183.183    www.xxx.com
```

 
浏览器访问https://www.xxx.com:8989