### Contents

- [CIS benchmarks](#cis-benchmarks)
- Authentication & Authorization mechanisms 
- [Service Accounts](#service-accounts)
- TLS 
- Node Metadata
- K8S Dashboard and securing it
- Platform Binaries
- Upgrade k8s
- Network policy 
- Secure ingress 


### CIS Benchmarks

* Security benchmark. For example, get to data server and infect via a usb device (physical access), so usb ports should be disable by default. 
* Access – who can access, and who can access as root. Sudo is configured and only certain users 
* Network – firewall rules, only allow certain port.  
* Services – only necessary services eg. Ntp, all others disabled.  
* Filesystem permissions – disable unneeded file system 
* Many more best practies.  
* CIS benchmarks allows us to assess our servers against these best practies.  
* Center for Inernet Security (CIS) 

* run assessment report
```
sh ./Assessor-CLI.sh -i -rd /var/www/html/ -nts -rp index

sh ./Assessor-CLI.sh -i -rd /var/www/html/ -nts -rp index
Use below setting while running tests

Benchmarks/Data-Stream Collections: : CIS Ubuntu Linux 18.04 LTS Benchmark v2.0.1

Profile : Level 1 - Server

```

* CIS for kubernetes
* Download from CIS Web Site
* https://workbench.cisecurity.org/
* kube-bench from Aqua Security
* Deploy as a docker container, or a pod or a binary.
* Download and install.
```
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.4.0/kube-bench_0.4.0_linux_amd64.tar.gz -o kube-bench_0.4.0_linux_amd64.tar.gz
tar -xvf kube-bench_0.4.0_linux_amd64.tar.gz
```
run kube-bench
```
./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml 
```

### Security Primitives

* kube-apiserver
* who can access and what can they do. 
* authenticate - usernames/password, certificate, external eg. ldap, service accounts
* authorization - rbac, groups and user permissions, abac etc.

Components (all secured by tls certs)
* Kube API Server
* ETCD
* Kubelet
* Kube Proxy
* Kube Scheduler
* Kube Controller Manager

By default all components can talk to each other, you can control comms by network policies.

### Authentication

- multiple nodes
- access by admin, developers, end users, 3rd party application for integration.
- focus is on admin users
- - users (human) 
- - robots
- k8s does not manage regular users
- k8s does manage service accounts

#### Users 
- all requests go through kube-apiserver, authenticates first and then processes
- users
- - static password file
- - static token file
- - certificates
- - identity services (ldap, kerberos etc.)

#### User File Authentication.
```
user-details.csv
password, user, id, group(optional)

kube-apiserver -basic-auth-file=user-details.csv
```

If using kubeadm
```
/etc/kubernetes/manifests/kube-apiserver.yaml

spec:
  containers:
  - command:
  - kube-apiserver
  ...
  - --basic-auth-file=user-details.csv
 ```
 
 you can then pass user/pass in a curl cmd eg.
 ```
 curl -v -k https://master-node-ip:6443/api/v1/pods -u "user1:pass"
 ```
 
 #### Token File Authentication
 ```
 user-token-details.csv
 kjjsskkkksi,user10,u0001,group1
 
 kube-apiserver --token-auth-file=user-token-details.csv
 
 curl -v -k https://master-node-ip:6443/api/v1/pods --header "Authorization: Bearer  kjjsskkkksi"
 ```
 
Note! : These two methods are considered insecure. 


### Service Accounts

User Accounts (humans)
Service Accounts (machines)

```
kubectl create serviceaccount dashboard-sa

kubectl get sa

```
Token is created automatically as a secrete using the name format
```
sa-name-token-<unique>
```

To view token
```
kubectl describe secret sa-name-token-unique
```
This token can then be used as a bearer token to the k8s api.
```
curl  -v -k https://master-node-ip:6443/api --header "Authorization: Bearer eyjhb...."
```

- A default service account is created automatically in every namespace
- A pod in that namespace automatically mounts that service account token secret
- /var/run/secretes/kubernetes.io/serviceaccount
- You can see this by running an ls on that location eg.
```
kubectl exec -it my-kubernetes-dashboard ls /var/run/secretes/kubernetes.io/serviceaccount
ca.crt namespace token
```
- token is in the file "token"
- default service account is restricted to running queries. 
- to use a account service account specify it in the pod
```
spec:
  containers:
  serviceAccount : dashboard-sa
```

### Service Account rbac
```
$ cat /var/rbac/pod-reader-role.yaml 
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - watch
  - list

cat /var/rbac/dashboard-sa-role-binding.yaml 
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: dashboard-sa # Name is case sensitive
  namespace: default
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io

```

Get the sa secret and token
```
k get secret
NAME                       TYPE                                  DATA   AGE
dashboard-sa-token-6jxkt   kubernetes.io/service-account-token   3      3m8s
default-token-jqwxm        kubernetes.io/service-account-token   3      177m
controlplane $ k describe secret dashboard-sa-token-6jxkt
Name:         dashboard-sa-token-6jxkt
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-sa
              kubernetes.io/service-account.uid: 4ad0ff43-6acf-44cf-9ce9-1f2225a444a5

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InVRamk0bGhqbzZsVGFFa21yU0FkZndCVWNoc1VqSHU4VFlfNmFZakRjbmcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRhc2hib2FyZC1zYS10b2tlbi02anhrdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkYXNoYm9hcmQtc2EiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI0YWQwZmY0My02YWNmLTQ0Y2YtOWNlOS0xZjIyMjVhNDQ0YTUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkYXNoYm9hcmQtc2EifQ.G1kZGXm8lp1YIlk_S0qgbgU6AaZCxiSq8QwF8UctixTfNt_XuYvsI_bYcMRu3glJKwMtE4YrEIlFMuk6SCjc45LkczzNvWTmPc05SlIukxTl5KPOcg2A-Ps-z58evcijUu-maoBu38v0AJSjsQNL4liionYZIFpkcy_KheoafXiERDJgHDvJmQNyIHpeWmQptIFWh13hXz1g-zavHdB0dy_Vn-QfuxzRvFwlasDgAK1-l00G61FD3qCYRBfqTr7qtRnoDQIqogBk7k4rzwOW0kOQgWUbNsxxMcyRfT3fFi9yIswOT0rJLsj__HjgLgXpF8mCyFgnCvM1IiS8zJqXDA
```


- Specify the service account in a deployment yaml eg.
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-deployment
spec:
  template:
    # Below is the podSpec.
    metadata:
      name: ...
    spec:
      serviceAccountName: build-robot
      automountServiceAccountToken: false
```


### TLS Certificate Basics

- certificates establish trust between parties.
- symmetric .v. asymetric encryption
- key pair - private key, public key. 
- Certificate Authorities


#### View Certificate Details
- healthcheck of certificates.
- "the hard way" .v. kubeadm
- thw = /etc/systemd/kube-apiserver.service
- kubeadm = /etc/kubernetes/manifests/kube-apiserver.yaml

##### Lab
- kube-api server cert file.
```
# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep crt
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    
    /etc/kubernetes/pki/apiserver.crt
    
```
- Identify the Certificate file used to authenticate kube-apiserver as a client to ETCD Server
```
# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    
    /etc/kubernetes/pki/apiserver-etcd-client.crt
```
- Identify the key used to authenticate kubeapi-server to the kubelet server
```
# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep kubelet
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    
    /etc/kubernetes/pki/apiserver-kubelet-client.key
```
- Identify the ETCD Server Certificate used to host ETCD server
```
# cat /etc/kubernetes/manifests/etcd.yaml | grep crt
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    
    /etc/kubernetes/pki/etcd/server.crt
```
- Identify the ETCD Server CA Root Certificate used to serve ETCD Server
```
/etc/kubernetes/pki/etcd/ca.crt
```
- What is the Common Name (CN) configured on the Kube API Server Certificate?
```
# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep CN
        Issuer: CN = kubernetes
        Subject: CN = kube-apiserver
        
        kube-apiserver
```
- What is the name of the CA who issued the Kube API Server Certificate?
```
# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep Issuer
        Issuer: CN = kubernetes
```
- Which of the below alternate names is not configured on the Kube API Server Certificate?
```
# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 1 Alternative
            X509v3 Subject Alternative Name: 
                DNS:controlplane, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:10.22.248.6

```
- What is the Common Name (CN) configured on the ETCD Server certificate?
```
# openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -text | grep CN
        Issuer: CN = etcd-ca
        Subject: CN = controlplane
        
        controlplane
```
- How long, from the issued date, is the Kube-API Server Certificate valid for?
```
# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 3 Validity
        Validity
            Not Before: Jul 24 15:52:33 2021 GMT
            Not After : Jul 24 15:52:34 2022 GMT
        Subject: CN = kube-apiserver
        
        1 year
```
- How long, from the issued date, is the Root CA Certificate valid for?
```
openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout | grep -A 3 Validity
        Validity
            Not Before: Jul 24 15:52:33 2021 GMT
            Not After : Jul 22 15:52:33 2031 GMT
        Subject: CN = kubernetes
        
        10 years
```
- Kubectl suddenly stops responding to your commands. Check it out! Someone recently modified the /etc/kubernetes/manifests/etcd.yaml file
```
cat /etc/kubernetes/manifests/etcd.yaml | grep crt
    - --cert-file=/etc/kubernetes/pki/etcd/server-certificate.crt
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
root@controlplane:~# ls /etc/kubernetes/pki/etcd/server-certificate.crt
ls: cannot access '/etc/kubernetes/pki/etcd/server-certificate.crt': No such file or directory
```

- The kube-api server stopped again! Check it out. Inspect the kube-api server logs and identify the root cause and fix the issue.
```
# kubectl get pods
Unable to connect to the server: net/http: TLS handshake timeout

# docker ps -a | grep apiserver
6955b6be9a89        ca9843d3b545           "kube-apiserver --ad…"   23 seconds ago       Exited (1) Less than a second ago                       k8s_kube-apiserver_kube-apiserver-controlplane_kube-system_b0c7fa5021cdd1d96107d4d0b0aa1b84_2
e3e5ee1254db        ca9843d3b545           "kube-apiserver --ad…"   About a minute ago   Exited (1) 36 seconds ago                               k8s_kube-apiserver_kube-apiserver-controlplane_kube-system_b0c7fa5021cdd1d96107d4d0b0aa1b84_1
2ed4ec96d244        k8s.gcr.io/pause:3.2   "/pause"                 About a minute ago   Up About a minute   

# docker logs 8f7dc5811bc2
W0724 16:24:39.766868       1 clientconn.go:1223] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379  <nil> 0 <nil>}. Err :connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority". Reconnecting...
W0724 16:24:39.775775       1 clientconn.go:1223] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379  <nil> 0 <nil>}. Err :connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority". Reconnecting...
W0724 16:24:40.771066       1 clientconn.go:1223] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379  <nil> 0 <nil>}. Err :connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority". Reconnecting...
W0724 16:24:41.364354       1 clientconn.go:1223] grpc: addrConn.createTransport failed to connect to {https://127.0.0.1:2379  <nil> 0 <nil>}. Err :connection error: des

/// port 2379 is used by ETCD

# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd
    - --etcd-cafile=/etc/kubernetes/pki/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    
/// edcd has its own ca
// change
- --etcd-cafile=/etc/kubernetes/pki/ca.crt
// to
- --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt


```

### Cert API

```
 cat /var/answers/akshay-csr.yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: akshay
spec:
  signerName: kubernetes.io/kube-apiserver-client
  groups:
  - system:authenticated
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZV3R6YUdGNU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQXd5OGpqQzdWNHVwdkRYT3ZRUDZkQVZMMG1yMHdmVmNlQ01EZXJnclBWRUIzClZxZEFrRmUzQW5uaCtycE5WcDVyRXNBb2xKZTQyWmhmRDFHc0N1OXBYcVVDSXJFSnpGNnVXWEhBdzFXbjNJRHoKWTJyNTlzMC9rY2ZaM0JyN203cGFNU0NnS1loZWVweSt3OCt3aXlPMWNNNkR4N2d3b2E5SHdDRWEwVzFqUlF3cApFL0dGWHB5TjhpTjlNMEJ3QjdJMjFaY093eFBBQVlkVUF0Uk1EcmtzNlZGanVha3dlam0vVjd4czF0NlZLNzMwCi9qWW9aYTdWRUJkZmY3VVhvSWErTDg1UzFDMUhLNnQ3am1VMUlYcHhOWjZTU0VJU3MyRVZDWkY3bExtMW01ZDIKc204a3J5WUhLQzhrQVdvMVlhSHRNMzVKN1FLY0ZtYlc2czlzTmVoWjB3SURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR3hxVDRybjRrd2ZiQW8waW5Rb0tQbURuMkh3dnhjOHZQOUlRYXlRaVJtb3E4cjNSVnNOClR1SDVXaHdKMEd0bVBLQ1U0dG01V1R0VXNVZGdCNmtHRzRIZTFNdEJBSlE2T3UwZU9jWjl1SldLbkNxMHBPRXMKNzhKa3BjY0lRdG9GSW56Y1ovZlQ2ZmVCc3JiRFh5bVQxU1M1MXk5Ly9jQnJKQjJzMFVYcVg4WnF5T0FmanpLRQppWVE1MTNsSUs2bjJiWXlaRXQ2Q3VDeXg4Q0s1d3hwYU1rNTVncnovNm5EYUZNR0ZXS3RhOFRrQ1VGT1JNQmxqCnNndXBRTVFEL2k3U2JUZk4rNlArZkJJMFVsck5LQWczYmRyendWWmhKVkF6ZDJIV3dqREc0VzQ3N2JZeEI2ek8KVkRrVGkyWnE0WGs2TURWdE8zeTEwRlROM2dteG9WZnRseGc9Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  usages:
  - digital signature
  - key encipherment
  - server auth
```

```
kubectl create -f /var/answers/akshay-csr.yaml

k get csr -A
NAME        AGE     SIGNERNAME                                    REQUESTOR                  CONDITION
akshay      27s     kubernetes.io/kube-apiserver-client           kubernetes-admin           Pending

kubectl certificate approve akshay

k get csr/agent-smith -o yaml | grep -A 2 groups:
  groups:
  - system:masters
  - system:authenticated

kubectl certificate deny agent-smith

 kubectl delete csr agent-smith

```

### Kubeconfig

- Access k8s api via cert
```
curl http://api-server:6443/api/v1/pods --key admin.key --cert admin.crt --cacert ca.crt
```
and via kubectl
```
kubectl get pods --server api-server:6443 --client-key admin.key --client-certificate admin.crt --certificate-authority ca.crt
```
or kubeconfig, by default in 
```
$HOME/.kube/config
```
or via cli
```
--kubeconfig config
```

Kubeconfig has 3 sections
- clusters
- - eg. production, dev, google
- contexts
- - eg admin@production, dev@google
- users
- - eg. Admin, DevUser, ProdUser

```
apiVersion: v1
kind: Config

clusters:
- name:
  cluster:
    certificate-authority:
    server:
    
contexts:
- name:
  context:
    cluser:
    user:

users:
- name:
  user:
    client-certificate:
    client-key:

```
- Current context is reflected in the file
```
kubectl config use-context my-context

apiVersion: v1
kind: Config
current-context: my-context
```

- A context can specify a namespace
```
    
contexts:
- name:
  context:
    cluser:
    user:
    namespace: my-namespace
```
- Certificates in kubeconfig should be a full path to the file, however you can also inline certificate information
```
# encode cert as base64
cat ca.crt | base64

clusters:
- name:
  cluster:
    certificate-authority-data: <base64-string>
    server:
    
# to decode
echo '<base64-string>' | base64 --decode
```

### API Groups
api's are organized into groups
```
/version
/api
/apis
/metrics
/healthz
/logs
```

- core group /api
- named group /apis
- going forward named groups will be used

hierarchy
```
named          
   - api group     eg. networking.k8s.io
     - resource   eg. networkpolicy
```

actions / verbs
- list, get, create, delete, update, watch 

- access the api via proxy
```
kubectl proxy

$ curl -k 127.0.0.1:8001
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/allspark.vmware.com",
    "/apis/allspark.vmware.com/v1alpha1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    ....
```
- List api groups
```
$ curl -k 127.0.0.1:8001/apis | jq .groups[].name

"apiregistration.k8s.io"
"apps"
"events.k8s.io"
"authentication.k8s.io"
"authorization.k8s.io"
"autoscaling"
"batch"
"certificates.k8s.io"
"networking.k8s.io"
"extensions"
"policy"
"rbac.authorization.k8s.io"
"storage.k8s.io"
"admissionregistration.k8s.io"
"apiextensions.k8s.io"
"scheduling.k8s.io"
"coordination.k8s.io"
"node.k8s.io"
"discovery.k8s.io"
"flowcontrol.apiserver.k8s.io"
"tsm.vmware.com"
"allspark.vmware.com"
"autoscaling.tsm.tanzu.vmware.com"
"client.cluster.tsm.tanzu.vmware.com"
"crd.antrea.tanzu.vmware.com"
"install.istio.io"
"kappctrl.k14s.io"
"ops.antrea.tanzu.vmware.com"
"security.antrea.tanzu.vmware.com"
"config.istio.io"
"core.antrea.tanzu.vmware.com"
"networking.istio.io"
"clusterinformation.antrea.tanzu.vmware.com"
"security.istio.io"
"stats.antrea.tanzu.vmware.com"
"controlplane.antrea.tanzu.vmware.com"
"metrics.k8s.io"
"networking.antrea.tanzu.vmware.com"
"system.antrea.tanzu.vmware.com"

```
- List storage resources
```
curl -k 127.0.0.1:8001/apis/storage.k8s.io/v1 | jq '.resources[].name'
"csidrivers"
"csinodes"
"storageclasses"
"volumeattachments"
"volumeattachments/status"
```

- List antrea resources
```
$ curl -k 127.0.0.1:8001/apis/networking.antrea.tanzu.vmware.com/v1beta1 | jq .resources[].name
"addressgroups"
"appliedtogroups"
"networkpolicies"
```
- Get etcd health
```
curl -k 127.0.0.1:8001/healthz/etcd
ok
```

### Authorization

- *node* authorizer. 
- Privileges used by kubelet to authorize to api-server
- *ABAC*
- policy file passed to api-server with the set of permissions (requires restart on every change)

- *RBAC*
- define roles
- assign roles to users / groups
- provides a more standardized approach.

- *Webhook*
- for 3rd party apps. 

- *Always Allow*
- *Always Deny*

Authorization mode is specified in the api-server cli params eg.
if a mode denies a request it is passed to the next mode in the chain
as soon as a mode approves a request no more checks are done. 
```
--authorization-mode=Node, RBAC, Webhook
```

### RBAC

- roles
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name:
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list","get","create"...]
```

- rolebindings
```
apiversion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io/v1
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io/v1
```

- check access
```
kubectl auth can-i create deployments

# as another user
kubectl auth can-i create deployments --as dev-user

# as another user in a specific namespace
kubectl auth can-i create deployments --as dev-user --namespace test
```


- you can restrict access to specific named resources by adding a resourceNames attribute to the role eg.
```
eg. access to only the "blue" and "green" pods.

rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list","get","create"...]
  resourceNames: ["blue","green"]
```

#### Lab

- check the access modes
```
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep mode
    - --authorization-mode=Node,RBAC
```
- How many roles exist in the default namespace?
```
kubectl get roles -n default
No resources found in default namespace.
```
- how many in all namespaces
```
kubectl get roles -A | wc -l
13
```
- What are the resources the kube-proxy role in the kube-system namespace is given access to?
```
kubectl get role kube-proxy -n kube-system -o yaml | grep resourceNames: -A 5
  resourceNames:
  - kube-proxy
  resources:
  - configmaps
  verbs:
  - get
```
- Check if the user can list pods in the default namespace.
```
k auth can-i list pods -n default --as dev-user
no
```
- Add permissions to create deployments
```
rules:
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["create"]
- apiGroups:
  - ""
  resourceNames:
  - dark-blue-app
  resources:
  - pods
  verbs:
  - get
  - watch
  - create
  - delete

# k auth can-i -n blue --as dev-user create deployment
yes
  
```

### Cluster Roles & Bindings

- some resources are namespace scoped eg. pods, jobs, secrets etc.
- some resources are cluster scoped, eg. nodes, pv, clusterrole, clusterrolebinding, csr, namespaces
- to see a list of namespaced resources
```
kubectl api-resources --namespaced=true
```
- to see a full list of cluster scoped resources
```
kubectl api-resources --namespaced=false
```
- cluster roles are roles for cluster scoped resources
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "create", "get","delete"]
```
- cluster role binding
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-admin-binding
subjects:
- kind: User
  apiGroup: rbac.authorization.k8s.io
  name: michelle
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: node-admin
```
- some default cluster roles


- example for storage resources
```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: storage-admin
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "watch", "list", "create", "delete"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "watch", "list", "create", "delete"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: michelle-storage-admin
subjects:
- kind: User
  name: michelle
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io
```

### Kubelet Security
- secure traffice between the api-server and the kubelet on each of the worker nodes.
role of the kubelet
- registers node
- creates pod via containerd / docker
- monitor node and pods and reports to api-server
- to see the options configured for the running kubelet
```
ps -aux | grep kubelet
```
- configuration is in 
```
/var/lib/kubelet/config.yaml
```
- security aspects of kubelet
- only respond to requests from api-server
- ports 10250, 10255 api
- by default allows anonymous access

authentication
- by default allows all requests to go through without authentication.
- to disable set 
```
--anonymous-auth=false
```
- can also be set in 
```
kubelet-config.yaml
```
- enable authentication via certifciate or bearer token.

- to make changes to kubelete and restart.
```
# check authentication
curl -sk https://localhost:10250/pods

# modify the config
nano /var/lib/kubelet/config.yaml

# restart kubelet
systemctl restart kubelet
```

- check metrics on anon port
```
curl -sk http://localhost:10255/metrics


### Kubectl Proxy

- proxy can be used unauthenticated because we have authenticatetd it via kubectl.
- you can manage the cluster via kubectl from anywhere
- or via curl, proxy allows you to make unauthenticatetd calls to localhost

kubectl proxy
curl -k http://127.0.0.1:8001
```
- proxy can also be used to access services running within the cluster
```
kubectl proxy
curl http://localhost:8001/api/v1/namespaces/default/services/nginx/proxy
```
- alternatively we can port forward
- port-forward takes a pod, service or replicaset as an argument
```
kubectl port-forward service/nginx 28080:80
curl http://localhost:28080/
```


### Kubernetes Dashboard
- view configs, secrets
- create applications etc.
- need to protect access to it.
- it was originally very open and was the target of cyber attacks (eg. at tesla the dashboard was left open)
- you can access a set of recommended configuration by applying
```
kubectl apply -f https://path-to-dashboard/recommneded.yaml
```
- dashboard servicde is clusterip by default to prevent open access
- to access you can create a proxy
```
kubectl proxy
```

## Dashboard Authentication Mechanisms
- token
```
create a service account user and get its token
be careful in what permissions you assign
```

- kubeconfig
```
upload kubeconig file
```

## Lab
- start proxy
```
# deploy dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
proxy --address=0.0.0.0 --disable-filter &
# access endpoint url
https://8001-port-29676855ea704cee.labs.kodekloud.com/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login

# get the token for a user
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

- Setup service account, roles and role bindings for dashboard
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF
```

- admin RoleBinding
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dashboard-admin-binding
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF

```
- list-namespace ClusterRoleBinding
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-list-namespace-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: list-namespace
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF
```


### Verify platform binaries before deploying

- Download binaries from https://github.com/kubernetes/kubernetes/releases
- verify
```
curl https://dl.k8s.io/v1.20.0/kubernetes.tar.gz -L -o kubernetes.gz
```
- gen a hash of file
```
shasum -a 512 kubernetes.gz # macos
sha512sum kubernetes.tar.gz

compare to the sha on page eg. https://github.com/kubernetes/kubernetes/releases/tag/v1.20.9
```

### API Versions

- each cluster has a specific version
```
kubectl get nodes
k get nodes
NAME                                  STATUS   ROLES                  AGE     VERSION
dev-cluster-2-control-plane-h7jxl     Ready    control-plane,master   7d21h   v1.20.5+vmware.1
dev-cluster-2-md-0-75f7fc7c4c-5xzjh   Ready    <none>                 7d21h   v1.20.5+vmware.1
dev-cluster-2-md-0-75f7fc7c4c-6cznw   Ready    <none>                 7d21h   v1.20.5+vmware.1
dev-cluster-2-md-0-75f7fc7c4c-l4fb5   Ready    <none>                 7d21h   v1.20.5+vmware.1
```
- version = major.minor.patch
- minor version every few months.
- patches are released more often with fixes
- these are stable releases
- but also alpha and beta. 
- components have their own versions eg. etcd. 
- look at release notes to see
  
  
### Upgrading a cluster

- api-server talks to all other components.
- so, they should be at or below the version of the api-server
- those components are controller-manager, kubelet, kube-proxy, kube-scheduler, kubectl

- api-server : X eg. 1.10
- controller-manager : X-1. eg. 1.9
- kube-scheduler: X-1
- kubelet : X-2
- kube-proxy : X-1
- kubectl : X+1 > X-1 eg. 1.11 - 1.9

- kubernetes supports the most recent 3 minor versions eg. 1.12, 1.11, 1.10
- can you go 1.10 -> 1.13 ... no
- recommend you only upgrade one minor version at a time.
  
- use the upgrade plan
```
# kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.19.0
[upgrade/versions] kubeadm version: v1.19.0
I0729 19:17:48.253147   32028 version.go:252] remote version is much newer: v1.21.3; falling back to: stable-1.19
[upgrade/versions] Latest stable version: v1.19.13
[upgrade/versions] Latest stable version: v1.19.13
[upgrade/versions] Latest version in the v1.19 series: v1.19.13
[upgrade/versions] Latest version in the v1.19 series: v1.19.13

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
kubelet     2 x v1.19.0   v1.19.13

Upgrade to the latest version in the v1.19 series:

COMPONENT                 CURRENT   AVAILABLE
kube-apiserver            v1.19.0   v1.19.13
kube-controller-manager   v1.19.0   v1.19.13
kube-scheduler            v1.19.0   v1.19.13
kube-proxy                v1.19.0   v1.19.13
CoreDNS                   1.7.0     1.7.0
etcd                      3.4.9-1   3.4.9-1

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.19.13
```
  

- upgrade with kubeadm
```
kubeadm upgrade plan
kubeadm upgrade apply
```
- first upgrade the master
- then upgrade the workers
- api-server, scheduler and controller-manager will be down briefly during the upgrade.
- workloads continue to work as normal. 
- upgrade the master first.
- then upgrade workers using one of these upgrade strategies
  1. all workers at once, causes downtime for workloads.
  2. one worker node at a time.
  3. add new nodes to the cluster, add one new node at a time and move workloads from old node to new node.
- process
```
kubeadm upgrade plan
```
- note kubelets must be upgraded manually as they are not managed by kubeadm.
```
# first upgrade kubeadm.
apt-get upgrade -y kubeadm=1.12.0-00
kubeadm upgrade apply v1.12.0
# note kubectl get nodes will still show old version as it reports the version of the kubelet
# so we must also upgrade kubelet
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet
# now running kubectl get nodes shows the new version.
```
  
- now upgrade the workers.
```
# first drain the node (will also cordon it)
kubectl drain node-1 

# ssh to worker and run
apt-get upgrade -y kubeadm=1.12.0-00
apt-get upgrade -y kubelet=1.12.0-00
kubeadm upgrade node config --kubelet-version v.12.0
systemctl restart kubelet
```
- then uncordon the node
```
kubectl uncordon node-1
```
  
### Network Policies

- ingress
- egress
- be default "allow all" all traffic between pods and servcies within the cluster
- example
```
web pod listens on 80
api pod listens on 5000
db pod listens on 3306
```
- network policy applies to pods.
- uses lables and selectors to apply policy
```
# applies to pods with the label role:db and only allows traffic from pods with the name api-pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
  matchLabels:
    role: db
policyTypes:
- Ingress
ingress:
- from:
  - podSelector:
    matchLabels:
      name: api-pod
  ports:
  - protocol: TCP
    port: 3306
```
- not all cni's support networkpolicies, for example flannel does not

- NOTE
- podSelector will select pods in all namespaces by default.
- if you want to select by namespace use a namespaceSelector eg.
```
ingress:
- from:
  - podSelector: 
    ...
    namespaceSelector:
      matchLabels:
        name: prod
```
- ipblock can be used to allow ingress from non-pod external servers eg.
```
ingress:
- from:
  ...
  - ipBlock:
      cidr: 192.168.5.10/32
```

### Lab 

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  - Ingress
  ingress:
    - {}
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306

  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080

  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

### Ingress 

- a nodeport makes a service available on a high port eg. 38080 on the nodes in the cluster
- a loadbalancer service will send a request to the underlying platform to create a loadbalancer.
- think of ingress as a layer 7 loadbalancer built into kubernetes



### Docker Service Configuration

```
systemctl start docker
systemctl status docker
systemctl stop docker

# manually
dockerd --debug
# docker listens on unix socket, docker daemon is only available on the same host. 
/var/run/docker.sock
# to access docker daemon from another host
dockerd --debug --host=tcp://192.168.1.10:2375
# now daemon is acceessible from another host eg.
export DOCKER_HOST=tcp://192.168.1.10:2375
docker ps
# but be very careful with this, especially if no internet host, no authentication. 
# to enable encryption (note the new port number)
--host=tcp://192.168.1.10:2376 --tls=true -tlscert=/var/docker/server.pem -tlskey=/var/docker/serverkey.pem
# these settings can be moved to a coniguration file. 
/etc/docker/daemon.json
{
  "debug" : true,
  "hosts" : ["tcp://192.168.1.10:2376"],
  "tls" : true,
  "tlscert" : "/var/docker/server.pem",
  "tlskey" : "/var/docker/serverkey.pem" 
}
```

### Securing the Docker Daemon
- with access to the docker daemon can delete applications, delete volumes, run their own applications or gain access to the host by running a privileged container.
- you must secure the docker host eg.
  - disable root users
  - who has access
  - disable password based authentication
  - stricly enfore ssh based auth
  - disable unused ports etc.

- we can enable remote access using the method above.
- make sure its only exposed on internal intterfaces
- and use certificates 
- from the client
```
export DOCKER_TLS=true
export DOCKER_HOST=tcp://x.x.x.x:2376
docker ps
```
or 
```
docker --tls=true ps
```
- but still no authentication
- to do so we add tlsverify and tlscacert
```
/etc/docker/daemon.json
{
  "debug" : true,
  "hosts" : ["tcp://192.168.1.10:2376"],
  "tls" : true,
  "tlscert" : "/var/docker/server.pem",
  "tlskey" : "/var/docker/serverkey.pem", 
  "tlsverify": " true,
  "tlscacert": "/var/docker/cacert.pem"
}
```
- we would generate client certificate (client.pem and clientkey.pem, signed by tlscacert) and share with client
- and then on the client
```
export DOCKER_TLS_VERIFY=true
export DOCKER_HOST=tcp://x.x.x.x:2376
docker --tlscert --tlskey --tlscacert ps
```
- alternatively we can drop the client cert and key into the users ~/.docker directory and the docker commadn will automatically pick them up.
- so remember tls alone does not authenticate.
