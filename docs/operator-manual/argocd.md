# Deploy Argo CD in Compliant Kubernetes

This guide gives you an introduction to Argo CD and how you can restrict its role to access a specific namespace and operate only on the target namespace. 

Argo CD follows the GitOps pattern of using Git repositories as the source of truth for defining the desired application state. Argo CD is implemented as a kubernetes controller which continuously monitors and compares the desired state of the application (as defined in the repository) against the current state of the application in the cluster (running as a pod). 


Setting up Argo CD consists of two parts: Installing the CRDS and argocd components, Configuring dex for authentication.

## Installing the CRDs

!!! tip 
     For more information, please read the official [documentation](https://argoproj.github.io/argo-cd/) to learn more.

Before starting, make sure you have [kubectl](https://kubernetes.io/docs/tasks/tools/) utility and a kubeconfig file exported:     

```bash
export KUBECONFIG=<path to kubeconfig>
```

Use the following command to install the CRDs.

```bash
kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable
```
and download the installation manifest, to edit and restrict the roles 

```bash
wget https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/namespace-install.yaml
```

## Configuring Dex for authentication

Modify the argocd-cm configmap and pass in the necessary values 

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
data:
  kustomize.buildOptions: --enable-alpha-plugins # if needed 
  url: https://argo.xxxx.se
  oidc.config: |
    name: Dex
    issuer: <dex-issuer-url>
    clientID: kubelogin
    clientSecret: <kubeloginsecret>
    cliClientID: dex-secret-argocd-cli
  resource.exclusions: |
    - apiGroups:
      - "*"
      kinds:
      - "PodSecurityPolicy"
      - "SecurityContextConstraints"
      clusters:
      - "*"
    - apiGroups:
      - ""
      kinds:
      - "PersistentVolume"
      clusters:
      - "*"
  resource.inclusions: |
    - apiGroups:
      - ""
      kinds:
      - ConfigMap
      - Endpoints
      - LimitRange
      - Secret
      - Namespace
      - PersistentVolumeClaim
      - Pod
      - PodTemplate
      - ReplicationController
      - ResourceQuota
      - ServiceAccount
      - Service
      clusters:
      - "*"
    - apiGroups:
      - "rbac.authorization.k8s.io/v1"
      kinds:
      - RoleBinding
      - Role
      clusters:
      - "*"
    - apiGroups:
      - "networking.k8s.io"
      - "extensions"
      - "discovery.k8s.io"
      - "cert-manager.io"
      - "batch"
      - "argoproj.io"
      - "apps"
      - "apiextensions.k8s.io"
      - "apiregistration.k8s.io"
      - "acme.cert-manager.io"
      kinds:
      - "*"
      clusters:
      - "*" 
```