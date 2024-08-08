# Deploying the DNS Cluster Add-on

In this lab you will deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery, backed by [CoreDNS](https://coredns.io/), to applications running inside the Kubernetes cluster.

## The DNS Cluster Add-on

> [!TIP]
> Install [helm](https://helm.sh/docs/intro/install/) if you haven't done so.

Deploy the `coredns` cluster add-on:

```
helm repo add coredns https://coredns.github.io/helm
helm --namespace=kube-system install coredns coredns/coredns --set service.clusterIP="10.32.0.10"
```

> output

```
NAME: coredns-1
LAST DEPLOYED: Thu Aug  8 08:00:59 2024
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CoreDNS is now running in the cluster as a cluster-service.

It can be tested with the following:

1. Launch a Pod with DNS tools:

kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools

2. Query the DNS server:

/ # host kubernetes
```

List the pods created by the `coredns` deployment:

```
kubectl get pods -n kube-system
```

> output

```
NAME                         READY   STATUS    RESTARTS   AGE
coredns-b45bb477f-kjg96      1/1     Running   0          16m
```

List the service created by the installation:

```
kubectl get svc -n kube-system
```

> output

```
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
coredns     ClusterIP   10.32.0.10   <none>        53/UDP,53/TCP   61m
```

## Verification

Create a `busybox` deployment:

```
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

List the pod created by the `busybox` deployment:

```
kubectl get pods -l run=busybox
```

> output

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```
kubectl exec -ti busybox -- nslookup kubernetes
```

> output

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```
