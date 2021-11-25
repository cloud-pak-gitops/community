<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Manage Secrets -- Sealed Secret](#manage-secrets----sealed-secret)
  - [Tutorial](#tutorial)
    - [Create a KIND cluster](#create-a-kind-cluster)
    - [Install sealedSecret](#install-sealedsecret)
    - [Install client-side tool](#install-client-side-tool)
    - [Create a sealed secret file](#create-a-sealed-secret-file)
    - [Apply the sealed secret](#apply-the-sealed-secret)
  - [Summary](#summary)
    - [SealedSecret solve the problem managing all k8s resources on git .](#sealedsecret-solve-the-problem-managing-all-k8s-resources-on-git-)
    - [Git organization](#git-organization)
    - [Limitation](#limitation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Manage Secrets -- Sealed Secret

## Tutorial
### Create a KIND cluster

```console
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.20.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```


### Install sealedSecret 

```console
$ helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
$ helm install sealed-secrets --namespace kube-system --version 1.16.1 sealed-secrets/sealed-secrets
```

### Install client-side tool 

```
# macos
$ brew install kubeseal

# linux
$ GOOS=$(go env GOOS)
$ GOARCH=$(go env GOARCH)
$ wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/kubeseal-$GOOS-$GOARCH
$ sudo install -m 755 kubeseal-$GOOS-$GOARCH /usr/local/bin/kubeseal
```

### Create a sealed secret file

**Note:** the use of `--dry-run` - this does not create a secret in your cluster

```console
$ kubectl create secret generic secret-name --dry-run --from-literal=foo=bar -o yaml | \
 kubeseal \
 --controller-name=sealed-secrets \
 --controller-namespace=kube-system \
 --format yaml > mysealedsecret.yaml
```

The file mysealedsecret.yaml is a commitable file , since sensitive info is already encrypted .

If you would rather not need access to the cluster to generate the sealed secret you can run

```console
$ kubeseal \
 --controller-name=sealed-secrets \
 --controller-namespace=kube-system \
 --fetch-cert > mycert.pem
```

to retrieve the public cert used for encryption and store it locally. You can then run 'kubeseal --cert mycert.pem' instead to use the local cert e.g.


```console
$ kubectl create secret generic secret-name --dry-run --from-literal=foo=bar -o yaml | \
kubeseal \
 --controller-name=sealed-secrets \
 --controller-namespace=kube-system \
 --format yaml --cert mycert.pem > mysealedsecret.yaml
```

### Apply the sealed secret

```console
$ kubectl create -f mysealedsecret.yaml

# check the decrypted secret 
$ kubectl get secret secret-name -o yaml
apiVersion: v1
data:
  foo: YmFy
kind: Secret
metadata:
  creationTimestamp: "2021-11-25T02:28:51Z"
  name: secret-name
  namespace: default
  ownerReferences:
  - apiVersion: bitnami.com/v1alpha1
    controller: true
    kind: SealedSecret
    name: secret-name
    uid: 82f34bc4-bb18-4c71-ae5a-5bd6479503e6
  resourceVersion: "2491"
  uid: 4a63601f-c4b2-46b3-9690-e6e98b980582
type: Opaque

$ echo YmFy |base64 -d
bar
```

## Summary

### SealedSecret solve the problem managing all k8s resources on git . 
The SealedSecret can be decrypted only by the controller running in the target cluster and nobody else (not even the original author) is able to obtain the original Secret from the SealedSecret. 

### Git organization

Since the credentials are encryped and decrypted by different key-pair , user will have different env config for different target cluster even for same credentials .

```console
env/
   dev/
      sealedsecretA.yaml
      sealedsecretB.yaml      
   staging/
      sealedsecretA.yaml
      sealedsecretB.yaml      
   production/
      sealedsecretA.yaml
      sealedsecretB.yaml      
```

### Limitation

- Eventually the credentials end up as a k8s secret , which use base64 encoding that , it likely fails to satisfy the standards of some enterprise clients . Comparing to bypass k8s secret mechanism and inject secrets directly into pods from Vault
- Always keep the priviate key safe .