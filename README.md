# ksops-poc, sops for Kubernetes

## Overview
sops for Kubernetes decrypts Kubernetes sops manifest files that can be securely stored along side your code. Extends the Kubernetes API by declaring special _customer resource definition_ that extend their Kubernetes counterparts `kind`; 
`Deployment=ConfigDeploymentSops`, `Ingress=ConfigIngressSops`, `Service=ConfigServiceSops` ...

### Getting started
Directions are intended for Mac OSX users

### Prerequisites
- install brew, golang 1.8+, dep, and sops
```
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
$ brew install go
$ brew install dep
$ brew install sops
```
- set your $GOPATH
```
$ export GOPATH="$HOME/go"
```

### Requirements
- Kubernetes 1.9+
- Kubebuilder 1.0.5+
-~~ Minikube~~

## Quickstart

### Warning
Uses the default pgp key provided with the github.com/mozilla/sops repo. It is advised to import your own pgp keys and mount securely

```
$ git clone git@github.com:jecho/ksops-test.git
$ cd ksops-test
$ make
$ make deploy
```

## Build

### Publishing and Deploying
```
$ docker login
$ export IMG=jechocnct/ksops-poc
$ make docker-build
$ make docker-push
```
Deploy into our cluster
```
$ make deploy
```

### Locality with Minikube

```
$ git clone git@github.com:jecho/ksops-test.git
$ cd ksops-test
$ dep ensure
$ make 
$ make install
```
Run in our local cluster
```
$ make run
```

### Verify
Verify that _ksops-test-system_ and _custom resource definitions_ are up and running
```
$ kubectl get all -n=ksops-test-system
NAME                                  READY   STATUS    RESTARTS   AGE
pod/ksops-test-controller-manager-0   1/1     Running   3          8m

NAME                                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/ksops-test-controller-manager-service   ClusterIP   10.110.39.36   <none>        443/TCP   17m

NAME                                             DESIRED   CURRENT   AGE
statefulset.apps/ksops-test-controller-manager   1         1         17m

$ kubectl get crd
NAME                                  CREATED AT
configdeploymentsops.mygroup.k8s.io   2018-12-17T21:50:44Z
configingresssops.mygroup.k8s.io      2018-12-17T22:47:57Z
configservicesops.mygroup.k8s.io      2018-12-17T21:50:44Z
```
If resources do not come up, bind a _cluster role binding_ to ksops-test-system:default
```
$ kubectl create clusterrolebinding kube-sops-admin \
$   --clusterrole=cluster-admin \
$   --serviceaccount=ksops-test-system:default
```

## Testing

Files will be encrypted in the sops standard as shown below. snippet of _ghost_deployment.yaml_

```
apiVersion: mygroup.k8s.io/v1beta1
kind: ConfigDeploymentSops
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: configdeploymentsops-sample
spec:
  manifest: |
    apiVersion: ENC[AES256_GCM,data:XEZLS/OKVA==,iv:N6o/g2EMb4oQsFN981uyq1wuXiG9cHM2D7KWLpf70bk=,tag:VVEMifJscE6y+GbIJsHpyA==,type:str]
    kind: ENC[AES256_GCM,data:yuEsTnj7DajaOQ==,iv:fxzwgGp57iEIMywYIxLDajdj1G5VcdDryQRrIjPKztQ=,tag:8A1GlO37Pj+Sm3MMxZuGuA==,type:str]
    metadata:
        name: ENC[AES256_GCM,data:z8fCmAJylx86d8YZ,iv:3yWOPJoUJrRAEe4L+5NXbs7USpzyGsgixu+UdmNcGUk=,tag:mdT79lema3gf5UvXnECcig==,type:str]
    spec:
        replicas: ENC[AES256_GCM,data:gQ==,iv:rDoSdFgE2UuSBHxyHrbU+FiCMCGjoJ8xyb/DBMz+Ojk=,tag:cMigwItqjaDCy0jNmvyklg==,type:int]
        selector:
            matchLabels:
                name: ENC[AES256_GCM,data:RPzgigA=,iv:bJzUKoFliPiw07GsyJUaspb+BMV/vGTMKHC3CpwRPnU=,tag:VSDzMGFDtOv/MP0Pz/c2GQ==,type:str]
                env: ENC[AES256_GCM,data:08kWxwa0oQ==,iv:MQgVcjpug4oiqWpwmuFXDBcYnYr82uhTZlE7YcS4+gQ=,tag:dyqGPdlpyxbcHjwl7vNUKQ==,type:str]
        template:
  ...
```
Running files as their respective `kind` CRDs will decrypt the resources and that Kubernetes can consume

To do this, run an instance of our `ghost` demo

```
$ kubectl create -f ghost_deployment.yaml
$ kubectl create -f ghost_svc.yaml
```

Verify that the deployment is healthy and running
```
$ kubectl get configdeploymentsops.mygroup.k8s.io
NAME                          CREATED AT
configdeploymentsops-sample   1h

$ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
ghost-deploy-5fc8f79f75-rcr65   1/1     Running   0          8m

$ kubectl get service
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
ghost-svc    NodePort    10.109.185.129   <none>        80:30180/TCP   8m
```

Retrieve the `minikube ip` and the assigned `node port` and reach through your web browser
```
$ NODE_PORT=$(kubectl get svc ghost-svc --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
$ echo http://$(minikube ip):${NODE_PORT}
```

## Usage

### Types
`ConfigDeploymentSops` = `Deployment`  
`ConfigServiceSops` = `Service`  
`ConfigIngressSops` = `Ingress`  

### Encrypting Files
Encrypt your resource file. For our example, we use the stock pgp key provided in mozilla/sops repo
```
$ git clone https://github.com/mozilla/sops.git
$ cd sops
$ gpg --import pgp/sops_functional_tests_key.asc
$ sops -e -i resource.yaml
```
>output
```
apiVersion: mygroup.k8s.io/v1beta1
kind: ConfigDeploymentSops
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: configdeploymentsops-sample
spec:
  manifest: |
    apiVersion: ENC[AES256_GCM,data:XEZLS/OKVA==,iv:N6o/g2EMb4oQsFN981uyq1wuXiG9cHM2D7KWLpf70bk=,tag:VVEMifJscE6y+GbIJsHpyA==,type:str]
    kind: ENC[AES256_GCM,data:yuEsTnj7DajaOQ==,iv:fxzwgGp57iEIMywYIxLDajdj1G5VcdDryQRrIjPKztQ=,tag:8A1GlO37Pj+Sm3MMxZuGuA==,type:str]
    metadata:
        name: ENC[AES256_GCM,data:z8fCmAJylx86d8YZ,iv:3yWOPJoUJrRAEe4L+5NXbs7USpzyGsgixu+UdmNcGUk=,tag:mdT79lema3gf5UvXnECcig==,type:str]
    spec:
        replicas: ENC[AES256_GCM,data:gQ==,iv:rDoSdFgE2UuSBHxyHrbU+FiCMCGjoJ8xyb/DBMz+Ojk=,tag:cMigwItqjaDCy0jNmvyklg==,type:int]
  ...
```

After a yaml resource has been encrypted, you select it's associated `kind` and toss the sops data in the manifest keypair _value_ in spec:
```
apiVersion: mygroup.k8s.io/v1beta1
kind: ConfigDeploymentSops  <--- kubernetes preemptive wrapper 
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: configdeploymentsops-sample
spec:
  manifest: |
    <--- encrypted sops data
```
