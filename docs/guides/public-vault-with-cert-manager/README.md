## Public Vault with Cert-Manager and Let's Encrypt

### Overview

A possible use case of Vault is to use it as a public-facing service on the Internet for a distributed and omogenous set of clients to consume.

In a case like this using Self-Signed certificates poses a challenge in terms of either distributing the CA to all clients or tick the _forbidden_ SKIP_TLS_VERIFY button.

In a case like this [Let's Encrypt](https://letsencrypt.org/) and [cert-manager](https://github.com/jetstack/cert-manager) can help to setup a fully valid SSL endpoint for our vault cluster in a proper automated fashion.

At the end of this guide we will have an _end to end_ encrypted vault service using publicly valid Certificates

### PreRequisites and Assumptions

Some assumptions must be made in this guide in order to avoid writing yet another _kubernetes the hard way_ guide

* You must have an already running cluster to which you can authenticate with _cluster admin_ privileges
* All examples will be written for a cluster running on GKE platform, with the right adjustment they will work on any other platform
* You must have an already configured DNS zone on GCP and a kuberntes secret containing the JSON private key for a serviceaccount with admin access to the DNS Zone.
  * We will call the DNS zone _super.me_
	* We will call the Secret _cluster-dnsadmin_ and the key _credentials.json_
* The cert-manager deployment will be minimal with no webhooks or cainjector pods
* The bank-vauls deployment will be minimal with no webhooks
* We will expose vault as a Service of type LoadBalancer which will create a Google TCP Loadbalancer

### Starting Point

TODO: kubectl get nodes and such 


### Install cert-manager

* Install CRDs
  `kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.11.0/deploy/manifests/00-crds.yaml`
* Create namespace
  `kubectl create namespace cert-manager`
* Create RBAC Resources
```
cat <<EOF| kubectl apply -f -
---
# Certificates controller role
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cert-manager-controller-certificates
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates", "certificates/status", "certificaterequests", "certificaterequests/status"]
    verbs: ["update"]
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates", "certificaterequests", "clusterissuers", "issuers"]
    verbs: ["get", "list", "watch"]
  # We require these rules to support users with the OwnerReferencesPermissionEnforcement
  # admission controller enabled:
  # https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#ownerreferencespermissionenforcement
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates/finalizers"]
    verbs: ["update"]
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["orders"]
    verbs: ["create", "delete", "get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---
# Challenges controller role
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cert-manager-controller-challenges
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
rules:
  # Use to update challenge resource status
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["challenges", "challenges/status"]
    verbs: ["update"]
  # Used to watch challenge resources
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["challenges"]
    verbs: ["get", "list", "watch"]
  # Used to watch challenges, issuer and clusterissuer resources
  - apiGroups: ["cert-manager.io"]
    resources: ["issuers", "clusterissuers"]
    verbs: ["get", "list", "watch"]
  # Need to be able to retrieve ACME account private key to complete challenges
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  # Used to create events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
  # HTTP01 rules
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["extensions"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "delete", "update"]
  # We require these rules to support users with the OwnerReferencesPermissionEnforcement
  # admission controller enabled:
  # https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#ownerreferencespermissionenforcement
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["challenges/finalizers"]
    verbs: ["update"]
  # DNS01 rules (duplicated above)
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
---

# ClusterIssuer controller role
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cert-manager-controller-clusterissuers
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["clusterissuers", "clusterissuers/status"]
    verbs: ["update"]
  - apiGroups: ["cert-manager.io"]
    resources: ["clusterissuers"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---
# ingress-shim controller role
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cert-manager-controller-ingress-shim
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates", "certificaterequests"]
    verbs: ["create", "update", "delete"]
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates", "certificaterequests", "issuers", "clusterissuers"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  # We require these rules to support users with the OwnerReferencesPermissionEnforcement
  # admission controller enabled:
  # https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#ownerreferencespermissionenforcement
  - apiGroups: ["extensions"]
    resources: ["ingresses/finalizers"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---

# Issuer controller role
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cert-manager-controller-issuers
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["issuers", "issuers/status"]
    verbs: ["update"]
  - apiGroups: ["cert-manager.io"]
    resources: ["issuers"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---

# Orders controller role
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: cert-manager-controller-orders
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
rules:
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["orders", "orders/status"]
    verbs: ["update"]
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["orders", "challenges"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["cert-manager.io"]
    resources: ["clusterissuers", "issuers"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["challenges"]
    verbs: ["create", "delete"]
  # We require these rules to support users with the OwnerReferencesPermissionEnforcement
  # admission controller enabled:
  # https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#ownerreferencespermissionenforcement
  - apiGroups: ["acme.cert-manager.io"]
    resources: ["orders/finalizers"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-edit
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates", "certificaterequests", "issuers"]
    verbs: ["create", "delete", "deletecollection", "patch", "update"]
---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-view
  labels:
    app: cert-manager
    app.kubernetes.io/name: cert-manager
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates", "certificaterequests", "issuers"]
    verbs: ["get", "list", "watch"]
EOF
```


