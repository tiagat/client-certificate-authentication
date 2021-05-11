### Goal

When client certificate authentication is turned on, the client HTTPS connection must submit with a valid cert that signed by the CA. Otherwise, the connection will be rejected.



### Install tools for PoC (MacOS)

```
$ brew install nginx
$ brew install cfssl
```


Default config path for nginx /usr/local/etc/nginx/nginx.conf


Test 
```
$ brew services start nginx
$ curl -I http://127.0.0.1:8080

HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Tue, 11 May 2021 12:53:22 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Apr 2021 15:14:07 GMT
Connection: keep-alive
ETag: "6075b53f-264"
Accept-Ranges: bytes

$ brew services stop nginx
```


Add SSL listner to Nginx:


/usr/local/etc/nginx/nginx.conf

```
    server {
       listen       443 ssl;
       server_name  localhost;

       ssl_certificate      /Users/tiagat/Projects/ssl-auth-poc/server.pem;
       ssl_certificate_key  /Users/tiagat/Projects/ssl-auth-poc/server-key.pem;

       ssl_client_certificate /Users/tiagat/Projects/ssl-auth-poc/ca.pem;
       ssl_verify_client on;

       location / {
           root   html;
           index  index.html index.htm;
       }
    }
```


### Create CA


ca-config.json
```
{
  "signing": {
      "default": {
          "expiry": "43800h"
      },
      "profiles": {
          "server": {
              "expiry": "43800h",
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ]
          },
          "client": {
              "expiry": "43800h",
              "usages": [
                  "signing",
                  "key encipherment",
                  "client auth"
              ]
          }
      }
  }
}
```

ca-csr.json
```
{
  "CN": "localhost",
  "hosts": [
    "localhost"
  ],
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [{
      "C": "US",
      "O": "Kubernetes",
      "OU": "Custom Root CA"
  }]
}
```


```
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```


### create SSL cert for Nginx 


request-server.json
```
{
  "CN": "localhost",
  "hosts": [
    ""
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
```


```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server -hostname="127.0.0.1" request-server.json | cfssljson -bare server
```


### create SSSL cert for client


request-client.json
```
{
  "CN": "client",
  "hosts": [
    ""
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
```

```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client -hostname="127.0.0.1" request-client.json | cfssljson -bare client
```



### Negative Test

```
$ curl -k https://127.0.0.1:443

<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.19.10</center>
</body>
</html>
```

### Positive Test 

```
$ curl -k --cert client.pem --key client-key.pem  https://127.0.0.1:443


<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

