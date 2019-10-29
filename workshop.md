
## Workshop

In this workshop we will deploy **2 apps and Traefik2** to show how to handle/configure ingress resources for **TCP** and **HTTP**.

We will:

- Deploy 2 apps in our cluster:
  - **whoami**
  - **tcp-go**

- Deploy **Traefik2 custom helm chart** as default ingress controller.
- Configure some **ingress resources to handle TCP, TCP with SNI and HTTP traffic** to our cluster apps.
- Test and show some interesting things about traefik2 and ingress
- All resources needed for this workshop are already included in this repository.

### Prerequisites

**Minikube installation**

### Create tls secrets from TLS certificates

We've just created **2 self-signed SSL certificates** for testing purposes:

```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop$ ll certs/
total 32
drwxr-xr-x 2 dmurga developers 4096 oct 24 12:03 ./
drwxr-xr-x 7 dmurga developers 4096 oct 25 09:04 ../
-rw-r--r-- 1 dmurga developers 7757 oct 24 15:50 secrets.yaml
-rw-r--r-- 1 dmurga developers 1119 oct 24 11:52 tcp.localhost.test.crt
-rw-r--r-- 1 dmurga developers 1675 oct 24 11:52 tcp.localhost.test.key
-rw-r--r-- 1 dmurga developers 1127 oct 24 11:55 whoami.localhost.test.crt
-rw-r--r-- 1 dmurga developers 1675 oct 24 11:55 whoami.localhost.test.key
```

1. Show certs:

We will show details about our self-signed certificate, it's important to note our DNS name.
```
❯openssl x509 -text -noout -in tcp.localhost.test.crt
```

2. Create base64 hash from cert/key:
   
Kubernetes handles secret's information as **base64** codified hashes, so... let's code our .crt and .key to base64:

```
❯cat whoami.localhost.test.key | base64 -w 0
```

Now we've got our certificates in base64 coding, we can create a manifest file to create a secret object of **kind TLS** in our cluster. We'll use this TLS-secret later when configuring ingress resources.

1. Create secrets/tls manifest :

```
apiVersion: v1
data:
tls.crt: ***
tls.key: ***
kind: Secret
metadata:
name: tcp-localhost-test
namespace: default
type: kubernetes.io/tls

---

apiVersion: v1
data:
tls.crt: ***
tls.key: ***
kind: Secret
metadata:
name: whoami-localhost-test
namespace: default
type: kubernetes.io/tls

```

4. Apply the manifest:

```
❯k apply -f secrets.yaml
```

5. Check the certs:

```
❯k get secrets -A

NAMESPACE         NAME                                             TYPE                                  DATA   AGE
default           authsecret                                       Opaque                                1      17h
default           default-token-mf2nm                              kubernetes.io/service-account-token   3      21h
default           tcp-localhost-test                               kubernetes.io/tls                     2      17h
default           whoami-localhost-test                            kubernetes.io/tls                     2      17h
kube-node-lease   default-token-7qm8s                              kubernetes.io/service-account-token   3      21h
(...)

❯k describe secrets whoami-localhost-test

Name:         whoami-localhost-test
Namespace:    kube-system
Labels:       <none>
Annotations:  
Type:         kubernetes.io/tls

Data
====
tls.crt:  1127 bytes
tls.key:  1675 bytes


```
### Deploy demoapps (whoami/go-echo-tcp/dummy-pod)

Right now we'll deploy the 2 demo apps. Additionally a dummy-pod (ubuntu based) will be also deployed just for testing/dedugging purposes:

1. Show deployment/service manifests

```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop/demoapps$ ls -lrtah
total 28K
-rw-r--r-- 1 dmurga developers 1,2K oct 24 12:15 go-echo-deploy.yaml
-rw-r--r-- 1 dmurga developers  400 oct 24 12:16 whoami_deployment.yaml
-rw-r--r-- 1 dmurga developers  172 oct 24 12:16 whoami_service.yaml
-rw-r--r-- 1 dmurga developers  239 oct 24 12:21 go-echo-service.yaml
drwxr-xr-x 2 dmurga developers 4,0K oct 24 12:57 .
-rw-r--r-- 1 dmurga developers  219 oct 24 14:12 dummy_pod.yaml
drwxr-xr-x 7 dmurga developers 4,0K oct 25 09:04 ..
```

2. Ensure namespace **= default!**

```
❯ cat whoami_deployment.yaml                
---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: whoami
  labels:
    app: whoami
```

3. Deploy deployments + services:

Let's deploy our apps:

```
❯k apply -f whoami_deployment.yaml

❯k apply -f whoami_service.yaml

❯k apply -f go-echo-deploy.yaml

❯k apply -f go-echo-service.yaml

❯k apply -f dummy_pod.yaml

```

4. Show deployed apps:
```
❯k get deployment -A

NAMESPACE     NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
default       tcp-echo-deployment    2/2     2            2           21h
default       whoami                 2/2     2            2           21h
(...)
```
```
❯k get services -A

NAMESPACE     NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                           AGE
default       kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP                           21h
default       tcp-echo-service       ClusterIP   10.102.70.118    <none>        2701/TCP                          21h
default       whoami                 ClusterIP   10.105.4.231     <none>        80/TCP                            21h
(...)

```
### Access demoapps through clusterIP

1. Attach to shell in dummy-pod pod:
```
❯k exec -it dummy-pod /bin/bash

```
2. Install tools:
```
❯apt update

❯apt install dnsutils

❯apt install curl

❯apt install telnet
```

3. Access services from dummy-pod
   
   We'll acces our apps from **inside** the cluster

   - Get Cluster Internal DNS name of our services:
     ```
	 	root@dummy-pod:/# nslookup 10.105.4.231
		231.4.105.10.in-addr.arpa	name = whoami.default.svc.cluster.local.
	 ```
	
    More info: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
	
	- Curl whoami app through ClusterIP service:
	 ```
	    root@dummy-pod:/# curl whoami.default.svc.cluster.local 
		Hostname: whoami-5f88d9c754-2pbhl
		IP: 127.0.0.1
		IP: 172.17.0.2
		RemoteAddr: 172.17.0.13:55710
		GET / HTTP/1.1
		Host: whoami.default.svc.cluster.local
		User-Agent: curl/7.58.0
		Accept: */*
	```
		Take note how we're loadbalanced between pods:

	```
		root@dummy-pod:/# curl whoami.default.svc.cluster.local                
		Hostname: whoami-5f88d9c754-2pbhl
		IP: 127.0.0.1
		IP: 172.17.0.2
		RemoteAddr: 172.17.0.13:56286
		GET / HTTP/1.1
		Host: whoami.default.svc.cluster.local
		User-Agent: curl/7.58.0
		Accept: */*

		root@dummy-pod:/# curl whoami.default.svc.cluster.local
		Hostname: whoami-5f88d9c754-jvrxs
		IP: 127.0.0.1
		IP: 172.17.0.8
		RemoteAddr: 172.17.0.13:56326
		GET / HTTP/1.1
		Host: whoami.default.svc.cluster.local
		User-Agent: curl/7.58.0
		Accept: */*
	```	

  ```
  ❯ k exec -it dummy-pod /bin/bash
  root@dummy-pod:/# telnet 172.17.0.6  2701^C
  root@dummy-pod:/# nslookup 10.99.113.66   
  66.113.99.10.in-addr.arpa	name = tcp-echo-service.default.svc.cluster.local.

  root@dummy-pod:/# telnet tcp-echo-service.default.svc.cluster.local. 2701
  Trying 10.99.113.66...
  Connected to tcp-echo-service.default.svc.cluster.local.
  Escape character is '^]'.
  Welcome, you are connected to node minikube.
  Running on Pod tcp-echo-deployment-7c8b99dbcd-8ws9d.
  In namespace default.
  With IP address 172.17.0.6.
  Service default.
  ^]
  telnet> quit
  Connection closed.
  root@dummy-pod:/# 
  ```

### Traefik2 Helm chart

Now that we've got our apps running let's **publish** to our users.

We'll deploy traefik2 with a helm chart that was developed within our Devops Team:

```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop/traefik-helm-chart$ tree
.
├── Chart.yaml
├── LICENSE
├── README.md
├── templates
│   ├── crd
│   │   ├── ingressroutetcp.yaml
│   │   ├── ingressroute.yaml
│   │   ├── middlewares.yaml
│   │   └── tlsoptions.yaml
│   ├── dashboard-service.yaml
│   ├── _helpers.tpl
│   ├── ingress-dashboard.yaml
│   ├── rbac
│   │   ├── clusterrolebinding.yaml
│   │   ├── clusterrole.yaml
│   │   └── serviceaccount.yaml
│   ├── traefik2-deployment.yaml
│   └── traefik2-service.yaml
└── values.yaml

```

1. Initialize helm:
```
❯ helm init   
```

2. Deploy the chart:
```
❯helm install --name traefik2 –debug

```
3. Get deployment logs:

```
❯kubectl -n kube-system logs -f deployment/traefik2 --all-containers=true --since=10m

```


### Test Traefik2 Ingresses
#### HTTP

For our tests we'll use the useful **"kubectl port-forward"** tool:

```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop/traefik-helm-chart$ kubectl port-forward --help
Forward one or more local ports to a pod. This command requires the node to have 'socat' installed.
```

1. Add entries in etc/hosts

```
#################### WORKSHOP #####################
127.0.0.1       tcp.localhost.test
127.0.0.1       whoami.localhost.test
127.0.0.1       dashboard.localhost.test
127.0.0.1       tcp-fake.localhost.test
###################################################
```
2. Redirect with kube

```
❯k port-forward service/traefik2 8080:30080 -n kube-system
```

3. Take a look at dashoboard ingress manifest and access **dashboard**:

```
# t2-dashboard-svc
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: t2-dashboard-svc
  namespace: kube-system
  annotations:
        "helm.sh/hook": post-install
spec:
  entryPoints:
  - http-ep
  routes:
  - kind: Rule
    match: Host (`dashboard.localhost.test`)
    services:
    - name: t2-dashboard-svc
      port:  80
```

```
http://dashboard.localhost.test:8080
```

4. Apply whoami ingress

```
❯ k apply -f ingress_whoami.yaml

❯ k get ingressroute.traefik.containo.us -A    

NAMESPACE     NAME               AGE
default       whoami-ingress     18h
kube-system   t2-dashboard-svc   18h

```

5. Access the URL:

```
http://whoami.localhost.test:8080/
```
#### HTTP - Middlewares

We'll create a middleware to provide whoami app with a user/password prompt:

1. We'll need to create a kubernetes secret of kind Opaque:

   - Create credentials by using htpasswd tool (admin/whoami):

  ```
  ❯ htpasswd -c credentials admin
  New password: 
  Re-type new password: 
  Adding password for user admin
  ```
   - Take a look at credentials:

  ```
	❯ cat credentials
  admin:$apr1$Ul6zX7Pr$kEEtp3M.NPeSiv8Smr95u0
  ```


2. Create a new secret for whoami:

```
kubectl create secret generic admin --from-file credentials
```

3. Create a new middleware type basic-auth:
```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop/ingresses/basic-auth$ cat whoami-middleware-basic-auth.yaml 
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: default
spec:
  basicAuth:
     secret: admin
```
```
❯ k apply -f whoami-middleware-basic-auth.yaml 
```
3. Delete old ingressroute
```
❯ k delete ingressroute whoami-ingress -n default           
ingressroute.traefik.containo.us "whoami-ingress" deleted
```

4. Create a new ingress definition whoami-ingress to add the middleware:
```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop/ingresses/basic-auth$ cat ingress_whoami_basic_auth.yaml 
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami-ingress-basic-auth
  namespace: default
spec:
  entryPoints:
    - http-ep
  routes:
  - match: Host(`whoami.localhost.test`)
    kind: Rule
    services:
      - name: whoami
        port: 80
    middlewares:
      - name: basic-auth
```
5. Apply the new ingress and test that if we access our app we're asked for a user/password.

#### TCP PLAIN

The most fancy Traefik2 improvement is that's able to handle **TCP traffic**.
In addition to the new TCP feature containous guys provided Traefik2 with SNI handling capabilities.

More info about SNI: https://www.cloudflare.com/learning/ssl/what-is-sni/

First we'll create an ingress resource without SNI handling

1. Create an ingressrouteTCP resource manifest:
```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop/ingresses$ cat ingress_go-echo.yaml 
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
   name: go-echo-ingress
   namespace: default
spec:
  entryPoints:
    - tcp-ep
  routes:
  - match: HostSNI(`*`)
    kind: Rule
    services:
    - name: tcp-echo-service
      port: 2701
```

2. Apply ingressroutetcp:

```
k apply -f ingress_go-echo.yaml
```
```
❯ k get ingressroutetcp.traefik.containo.us -A
NAMESPACE   NAME                  AGE
default     go-echo-ingress       6s
```
3. Forward local traffic to our tcp-ep entry-point:

```
❯ k port-forward service/traefik2 8081:31521 -n kube-system
Forwarding from 127.0.0.1:8081 -> 31521
```


4. Test access from outside the cluster:

```
❯ telnet tcp.localhost.test 8081                                 
Trying 127.0.0.1...
Connected to tcp.localhost.test.
Escape character is '^]'.
Welcome, you are connected to node minikube.
Running on Pod tcp-echo-deployment-7c8b99dbcd-c6tnv.
In namespace default.
With IP address 172.17.0.3.
Service default.
```

We're able to access our TCP service from the outside... but note that the rule that we're applying to our ingress route is  **HostSNI(`*`)** so... we can access our app without target hostname restrictions...

```
❯ telnet tcp-fake.localhost.test 8081 
Trying 127.0.0.1...
Connected to tcp-fake.localhost.test.
Escape character is '^]'.
Welcome, you are connected to node minikube.
Running on Pod tcp-echo-deployment-7c8b99dbcd-c6tnv.
In namespace default.
With IP address 172.17.0.3.
Service default.
```

#### TCP SSL

To be able to handle to which hostname we want to connect we'll need to enable SSL capabilities that will provide us **SNI**

1. Delete ingressroutetcp
```
❯ k get ingressroutetcp.traefik.containo.us -A                     
NAMESPACE   NAME              AGE
default     go-echo-ingress   8m17s

❯ k delete ingressroutetcps.traefik.containo.us go-echo-ingress    
ingressroutetcp.traefik.containo.us "go-echo-ingress" deleted
```

2. Add new ingress:
```
dmurga@gratefuldead:~/git_code/traefik2/traefik2/ingress_workshop/ingresses$ cat ingress_go-echo-ssl.yaml 
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
   name: go-echo-ingress-ssl
   namespace: default
spec:
  entryPoints:
    - tcp-ep
  routes:
  - match: HostSNI(`tcp.localhost.test`)
    kind: Rule
    services:
    - name: tcp-echo-service
      port: 2701
  tls:
    secretName: tcp-localhost-test

```
**Note: tls and SNI settings!!!!**

```
k apply -f ingress_go-echo-ssl.yaml
```

4. Test connection:

```
❯ openssl s_client -connect tcp.localhost.test:8081

CONNECTED(00000005)
depth=0 CN = tcp.localhost.test
verify error:num=18:self signed certificate
verify return:1
depth=0 CN = tcp.localhost.test
verify return:1
---
Certificate chain
 0 s:CN = tcp.localhost.test
   i:CN = tcp.localhost.test
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIDDTCCAfWgAwIBAgIJANlKIHO7OO51MA0GCSqGSIb3DQEBBQUAMB0xGzAZBgNV
BAMMEnRjcC5sb2NhbGhvc3QudGVzdDAeFw0xOTEwMjQwOTUyMjRaFw0yOTEwMjEw
OTUyMjRaMB0xGzAZBgNVBAMMEnRjcC5sb2NhbGhvc3QudGVzdDCCASIwDQYJKoZI
hvcNAQEBBQADggEPADCCAQoCggEBALin6W8Luui0o0oXBLSgGV/U74T44jKKRnbM
fke4DetyKXAjuaSCjb1CUgBgNP0w1I9ZMNEwMmFhM2SC3Mlkxo05ZDywm/8bqVAt
Sc7VeHWLgQ/Pt/AT9hHvFeZYHXVy0IR8KLVql5hSeRHfzKs81Egdy8lIpkHGT2XR
RKezDji+gikUYKkwcZHKiSc2e8zbCctgTDXbipkTcBZ3JTEFn1raDOlaok1y6VsO
XF2MUiukWcyRZLUGKjLB7gbxWPxYmehJRZsK00cLgDV4qwQaEd0S09tf7jdcrz8o
/yX5Gd4IS2DktJ3GcBCw7hhZUBZJbNojXXI6V4IyQg0wmQKIk5kCAwEAAaNQME4w
HQYDVR0OBBYEFNAYM52cgDsA4tD55cdQd8otWCsQMB8GA1UdIwQYMBaAFNAYM52c
gDsA4tD55cdQd8otWCsQMAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQEFBQADggEB
ACJDB8sUkZ3qFoW48wUvdqKr25eQLUzyv6OepOJP7LLog0IqRqcaQDb4rzzEpE1p
sbUm6arpmAAxcjfPe5Y/uJ0/yHeBL8okfs8kUO8zRmU4hOZ/SgNH2U7i5CR0HNiw
ft60D2dZoQFBpb2O+mssTvqTRaWNVwnHfU8/cubnykSsSIp8vRvKs2nPcuWi6hlk
WZ158lofKtIL42hvWvqvrVnA4OWoxpTe3ZJE8+fPoH1pgH/5b2PL0Jsbc7QiXICe
DZJ1fahM2HB0RWoCBMsj08JRVKcIohRK0/BRWGPFa/G2JzxLC88k1UE/dM/kJIRH
lL9CwZwzzosWEfva2YHhdjw=
-----END CERTIFICATE-----
subject=CN = tcp.localhost.test

issuer=CN = tcp.localhost.test

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1341 bytes and written 400 bytes
Verification error: self signed certificate
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 18 (self signed certificate)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: FEF28B98F3195BDD6BE43A69986D4C0834D622277DB05A49DA9F56404D9EEA43
    Session-ID-ctx: 
    Resumption PSK: B1A5C06BFA418EE4AF9F8F6FF6FBB622E2437FA5492E4BD0A9F21A0E6941D101D6B9A988B9763A7D90DA7EFDAF3545C1
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 604800 (seconds)
    TLS session ticket:
    0000 - 19 c5 d2 75 d5 fd 5c 1f-a3 ac 3b 6b 09 be 5d 5b   ...u..\...;k..][
    0010 - fe 99 14 4f 6b cf 6b 40-3a 48 75 6b f0 91 fd 34   ...Ok.k@:Huk...4
    0020 - 11 be d0 81 bc 0d db e7-cf d2 b4 ce 0d 7d 50 87   .............}P.
    0030 - 24 80 c5 93 cd 23 8b 2e-87 a6 d3 fc d1 87 16 c8   $....#..........
    0040 - d9 70 b5 73 6f 63 9f 03-94 13 fd c6 f0 06 b2 9f   .p.soc..........
    0050 - 61 44 97 b1 50 98 d2 d2-80 6f 45 84 02 dd e8 97   aD..P....oE.....
    0060 - 9e 5c 6d 4d d2 8e 2e de-7d bf c2 72 58 ae 05 3c   .\mM....}..rX..<
    0070 - f6 eb 98 ef 67 ec ee 7b-50 7f 31 d8 86 b4 70 90   ....g..{P.1...p.
    0080 - b9                                                .

    Start Time: 1571991786
    Timeout   : 7200 (sec)
    Verify return code: 18 (self signed certificate)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
Welcome, you are connected to node minikube.
Running on Pod tcp-echo-deployment-7c8b99dbcd-c6tnv.
In namespace default.
With IP address 172.17.0.3.
Service default.
```

5. What if we access with non allowed URL?:

```
❯  openssl s_client -connect tcp-fake.localhost.test:8081
CONNECTED(00000005)
depth=0 CN = TRAEFIK DEFAULT CERT
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 CN = TRAEFIK DEFAULT CERT
verify error:num=21:unable to verify the first certificate
verify return:1
---
Certificate chain
 0 s:CN = TRAEFIK DEFAULT CERT
   i:CN = TRAEFIK DEFAULT CERT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIDRjCCAi6gAwIBAgIQFRDVdOXWH8LE2OPP9eb2wzANBgkqhkiG9w0BAQsFADAf
MR0wGwYDVQQDExRUUkFFRklLIERFRkFVTFQgQ0VSVDAeFw0xOTEwMjUwODI1MTJa
Fw0yMDEwMjQwODI1MTJaMB8xHTAbBgNVBAMTFFRSQUVGSUsgREVGQVVMVCBDRVJU
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAnujTl5B4yJFk8WpSNB79
Ji8XTgvnf0ryrLQbLUGAUdkmi6YsQB+WqM1NZEMTaY2MSbfAJO3cuBWpwR+UruPK
OtKgjz/tYpJGa0O38Hszs/gyNW48NJCqf0+ajFHFYH+3CeiMq6kArD5oMyX2G7/9
kF4U6F1XeAAoc6BeKl/gqXYVcZbvMMtDClrc6MtoimdEWi1NIP49ZtTLVLs/swaa
JnUCSIZPKc5gvyyrhWbL18sb6OhZ+hOmOIudoAP2yl5itgmJ9h9ndR/zRaednwmM
9dov+vjwNnSRFJlo7qoLIYw4R4qsHgZoxKz96EqpmxfQhVS8juxgMLT2FHLkz7Ew
twIDAQABo34wfDAOBgNVHQ8BAf8EBAMCA7gwDAYDVR0TAQH/BAIwADBcBgNVHREE
VTBTglE5NTZkMzc1OGY3MTI1YTQ1NDQ1Mjg5ZTMyNmUxMjAyZS45ZTI0YjM5M2Ri
NjVmZDJlYzUyOGIwNzdmMjA4ZDZmZS50cmFlZmlrLmRlZmF1bHQwDQYJKoZIhvcN
AQELBQADggEBAIbydAqkyJB5ELKMSV338hNToC8lwsVsyQDxGOSGzeoGnI5uSD9d
H88zJMpivZFG86cxlZ7ssqWBFKzsf9HfYlKJKL1f7X2YsrqykRrdRijAe1VfsHS2
n6v0vz1sOHwgoE7anvynnTMXIlmgMrEkg7CpHbZwXqklW/Vjw5CNygRlcyKQoZfE
WAOazayhPaCkVn88IY5oddvpcLUztFNTa+5aRRc0hsvfFtKklxi8wey/2i7lwqnd
xOOZj2s+1Q90wC82iLhpQEl/H9i4yUKt/gg/SYIfeZi9Stjn77wmG+r+DtkpCpgH
HDCNW0luG9IZzjE163lZMD4RVbM16nZar7s=
-----END CERTIFICATE-----
subject=CN = TRAEFIK DEFAULT CERT

issuer=CN = TRAEFIK DEFAULT CERT

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1398 bytes and written 405 bytes
Verification error: unable to verify the first certificate
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 21 (unable to verify the first certificate)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: 8C815930BECCD9D6AD2DD7041892848A81020904E8B565EFFA271960063E5CC6
    Session-ID-ctx: 
    Resumption PSK: 94C6B0699A0EC66E77E442E39735121D147043E386CA1CAA11A9B53FB8FE611B5EDCB4F2BCC3BAD3C548B6E1B66DC607
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 604800 (seconds)
    TLS session ticket:
    0000 - c0 47 de 21 32 24 00 10-f6 7b c7 13 17 52 c3 8c   .G.!2$...{...R..
    0010 - a9 d3 43 bc bf d9 72 db-ac 1a f9 ce 80 79 c5 37   ..C...r......y.7
    0020 - 79 2f 57 ff 94 e3 f8 2b-e2 4f 5c af ec e6 19 51   y/W....+.O\....Q
    0030 - c2 ff 30 82 5f ca 2f 94-3f 6a ca 2a 62 09 7d be   ..0._./.?j.*b.}.
    0040 - 8f 14 47 ea cd e3 42 c1-73 aa e6 08 4c 10 6d ea   ..G...B.s...L.m.
    0050 - c2 a8 e4 72 5f 03 fa 90-55 01 ef d8 1e ed 1c 72   ...r_...U......r
    0060 - 07 38 80 9a 6a b4 32 72-bd ce 84 d0 25 e5 e6 ae   .8..j.2r....%...
    0070 - 19 65 8f d0 56 82 dd f8-17 4f 7f 4f dd b7 3c 07   .e..V....O.O..<.
    0080 - 0f                                                .

    Start Time: 1571991913
    Timeout   : 7200 (sec)
    Verify return code: 21 (unable to verify the first certificate)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
``` 