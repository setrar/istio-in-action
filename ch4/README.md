# ch4

```
k -n istioinaction apply -f ch4/coolstore-gw.yaml
```
> Return
```
gateway.networking.istio.io/coolstore-gateway created
```

```
istioctl proxy-config \
         --namespace istio-system \
         listener deploy/istio-ingressgateway
```
> Return
```
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 8080  ALL   Route: http.8080
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*
```

```
istioctl proxy-config \
         --namespace istio-system \
         route deploy/istio-ingressgateway \
         --output json --name http.8080
```
> Return
```
[
    {
        "name": "http.8080",
        "virtualHosts": [
            {
                "name": "blackhole:80",
                "domains": [
                    "*"
                ]
            }
        ],
        "validateClusters": false
    }
]
```

```
k apply -f ch4/coolstore-vs.yaml 
```
> Return
```
virtualservice.networking.istio.io/webapp-vs-from-gw created
```

```
istioctl proxy-config \
         --namespace istio-system \
         route deploy/istio-ingressgateway
```
> Return
```
NAME          DOMAINS                     MATCH                  VIRTUAL SERVICE
http.8080     webapp.istioinaction.io     /*                     webapp-vs-from-gw.istioinaction
              *                           /stats/prometheus*     
              *                           /healthz/ready*        
```

```
istioctl proxy-config \
         --namespace istio-system \
         route deploy/istio-ingressgateway \
         --output json --name http.8080
```
> Return
```
[
    {
        "name": "http.8080",
        "virtualHosts": [
            {
                "name": "webapp.istioinaction.io:80",
                "domains": [
                    "webapp.istioinaction.io",
                    "webapp.istioinaction.io:*"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|80||webapp.istioinaction.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxGrpcTimeout": "0s"
                        },
                        "metadata": {
                            "filterMetadata": {
                                "istio": {
                                    "config": "/apis/networking.istio.io/v1alpha3/namespaces/istioinaction/virtual-service/webapp-vs-from-gw"
                                }
                            }
                        },
                        "decorator": {
                            "operation": "webapp.istioinaction.svc.cluster.local:80/*"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            }
        ],
        "validateClusters": false
    }
]

```

```
kubectl apply -f services/catalog/kubernetes/catalog.yaml 
```
> Return
```
serviceaccount/catalog unchanged
service/catalog created
deployment.apps/catalog created
```

```
kubectl apply -f services/webapp/kubernetes/webapp.yaml 
```
> Return
```
serviceaccount/webapp unchanged
service/webapp created
deployment.apps/webapp created
```

- [ ] Grab LB URL and IP address

   First IP address from LB seem to work better (i.e. sed '1d')

```
URL=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
IP=$(dig ${URL} +short | sed '1d')
```

```
curl http://${URL}/api/catalog
```

```
curl http://${URL}/api/catalog -H "Host: webapp.istioinaction.io"
```
> Return
```
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}]
```

```
kubectl create -n istio-system secret tls webapp-credential \
--key ch4/certs/3_application/private/webapp.istioinaction.io.key.pem \
--cert ch4/certs/3_application/certs/webapp.istioinaction.io.cert.pem
```
> Return
```
secret/webapp-credential created
```

```
brew install curl-openssl
```

```
curl https://${URL}/api/catalog -v -H "Host: webapp.istioinaction.io"
```
> Return
```
*   Trying 3.99.2.141:443...
* Connected to aed1f19c407c74379bda3504355fc3c1-616782180.ca-central-1.elb.amazonaws.com (3.99.2.141) port 443 (#0)
* ALPN: offers h2
* ALPN: offers http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to aed1f19c407c74379bda3504355fc3c1-616782180.ca-central-1.elb.amazonaws.com:443 
* Closing connection 0
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to aed1f19c407c74379bda3504355fc3c1-616782180.ca-central-1.elb.amazonaws.com:443 
```

```
curl -v -4 -H "Host: webapp.istioinaction.io" \
   https://webapp.istioinaction.io/api/catalog \
   --cacert ch4/certs/2_intermediate/certs/ca-chain.cert.pem \
   --resolve webapp.istioinaction.io:443:${IP}
```
> Return
```
* Added webapp.istioinaction.io:443:3.99.2.141 to DNS cache
* Hostname webapp.istioinaction.io was found in DNS cache
*   Trying 3.99.2.141:443...
* Connected to webapp.istioinaction.io (3.99.2.141) port 443 (#0)
* ALPN: offers h2
* ALPN: offers http/1.1
*  CAfile: ch4/certs/2_intermediate/certs/ca-chain.cert.pem
*  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN: server accepted h2
* Server certificate:
*  subject: C=US; ST=Denial; L=Springfield; O=Dis; CN=webapp.istioinaction.io
*  start date: Jul  4 12:49:32 2021 GMT
*  expire date: Jun 29 12:49:32 2041 GMT
*  common name: webapp.istioinaction.io (matched)
*  issuer: C=US; ST=Denial; O=Dis; CN=webapp.istioinaction.io
*  SSL certificate verify ok.
* Using HTTP2, server supports multiplexing
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* h2h3 [:method: GET]
* h2h3 [:path: /api/catalog]
* h2h3 [:scheme: https]
* h2h3 [:authority: webapp.istioinaction.io]
* h2h3 [user-agent: curl/7.83.0]
* h2h3 [accept: */*]
* Using Stream ID: 1 (easy handle 0x13a014e00)
> GET /api/catalog HTTP/2
> Host: webapp.istioinaction.io
> user-agent: curl/7.83.0
> accept: */*
> 
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
< HTTP/2 200 
< content-length: 357
< content-type: application/json; charset=utf-8
< date: Sun, 08 May 2022 03:41:07 GMT
< x-envoy-upstream-service-time: 23
< server: istio-envoy
< 
* Connection #0 to host webapp.istioinaction.io left intact
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}]
```

```
curl -v -4 -H "Host: webapp.istioinaction.io" \
    http://webapp.istioinaction.io/api/catalog  \
    --resolve webapp.istioinaction.io:80:${IP}
```
> Return
```
* Added webapp.istioinaction.io:80:3.99.2.141 to DNS cache
* Hostname webapp.istioinaction.io was found in DNS cache
*   Trying 3.99.2.141:80...
* Connected to webapp.istioinaction.io (3.99.2.141) port 80 (#0)
> GET /api/catalog HTTP/1.1
> Host: webapp.istioinaction.io
> User-Agent: curl/7.83.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 357
< content-type: application/json; charset=utf-8
< date: Sun, 08 May 2022 03:48:41 GMT
< x-envoy-upstream-service-time: 7
< server: istio-envoy
< 
* Connection #0 to host webapp.istioinaction.io left intact
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}]
```

- [ ] Adding TLS Redirect

```
curl -v -4 -H "Host: webapp.istioinaction.io" \
http://webapp.istioinaction.io/api/catalog  \
--resolve webapp.istioinaction.io:80:${IP}
```
> Return
```
* Added webapp.istioinaction.io:80:3.99.2.141 to DNS cache
* Hostname webapp.istioinaction.io was found in DNS cache
*   Trying 3.99.2.141:80...
* Connected to webapp.istioinaction.io (3.99.2.141) port 80 (#0)
> GET /api/catalog HTTP/1.1
> Host: webapp.istioinaction.io
> User-Agent: curl/7.83.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 301 Moved Permanently
< location: https://webapp.istioinaction.io/api/catalog
< date: Sun, 08 May 2022 03:57:24 GMT
< server: istio-envoy
< content-length: 0
< 
* Connection #0 to host webapp.istioinaction.io left intact
```

## :x: Make sure certificates are present at the required location

```
curl -v -4 -H "Host: webapp.istioinaction.io" \
   https://webapp.istioinaction.io/api/catalog \
   --cacert ch4/certs/2_intermediate/certs/ca-chain.cert.pem \
   --resolve webapp.istioinaction.io:443:${IP}
```
> Return
```
* Added webapp.istioinaction.io:443:3.99.2.141 to DNS cache
* Hostname webapp.istioinaction.io was found in DNS cache
*   Trying 3.99.2.141:443...
* Connected to webapp.istioinaction.io (3.99.2.141) port 443 (#0)
* ALPN: offers h2
* ALPN: offers http/1.1
* error setting certificate verify locations:  CAfile: ch4/certs/2_intermediate/certs/ca-chain.cert.pem CApath: none
* Closing connection 0
curl: (77) error setting certificate verify locations:  CAfile: ch4/certs/2_intermediate/certs/ca-chain.cert.pem CApath: none
```

```
k apply -f ch4/echo.yaml 
```
> Return
```
deployment.apps/tcp-echo-deployment created
service/tcp-echo-service created
```

```
k get svc -n istio-system istio-ingressgateway -o json | jq '.spec.ports[] | select ( .name | contains("tcp"))'
```
> Return
```
{
  "name": "tcp",
  "nodePort": 30876,
  "port": 31400,
  "protocol": "TCP",
  "targetPort": 31400
}
```

```
k apply -f ch4/echo-vs.yaml 
```
> Return
```
virtualservice.networking.istio.io/tcp-echo-vs-from-gw created
```


```
URL=$(k get svc -n istio-system istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

```
telnet ${URL} 31400
```
> Return
```
Trying 3.99.2.141...
Connected to aed1f19c407c74379bda3504355fc3c1-616782180.ca-central-1.elb.amazonaws.com.
Escape character is '^]'.
Welcome, you are connected to node ip-172-20-33-16.ca-central-1.compute.internal.
Running on Pod tcp-echo-deployment-584f6d6d6b-dvqhz.
In namespace istioinaction.
With IP address 100.96.1.35.
Service default.
by now
by now
```

```
istioctl install -y -n istioinaction -f ch4/my-user-gateway.yaml
```
> Return
```
Ingress gateways installed                                                                                                
Installation complete
Making this installation the default for injection and validation.

Thank you for installing Istio 1.13.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/pzWZpAvMVBecaQ9h9
```

```
k apply -f ch4/my-user-gw-injection.yaml                                         
```
> Return
```
deployment.apps/my-user-gateway-injected created
service/my-user-gateway-injected created
role.rbac.authorization.k8s.io/my-user-gateway-injected-sds created
rolebinding.rbac.authorization.k8s.io/my-user-gateway-injected-sds created
```
