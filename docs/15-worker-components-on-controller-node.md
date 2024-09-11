## Deploying Worker Components on the Controller Node

This guide outlines the process of deploying Kubernetes worker components (kubelet, kube-proxy) onto the controller node. This can be necessary for scenarios like setting up cluster networking using CNI plugins, as these plugins often require worker components to function correctly on all nodes, including the controller.

**Why do this?**

Without worker components on the controller node, it may not be able to join the cluster network, resulting in in-cluster services being unavailable to crucial controller plane components like the API Server.

## Prerequisites

Assume you already have a running kubernetes instance.

Copy the appropriate certificates and private keys to the `server` machine:

```bash
for host in server; do
  ssh root@$host mkdir /var/lib/kubelet/
  
  scp ca.crt root@$host:/var/lib/kubelet/
    
  scp admin.crt \
    root@$host:/var/lib/kubelet/kubelet.crt
    
  scp admin.key \
    root@$host:/var/lib/kubelet/kubelet.key
done
```

Copy the `kubelet` and `kube-proxy` kubeconfig files to the `server` instance:

```bash
for host in server; do
  ssh root@$host "mkdir /var/lib/{kube-proxy,kubelet}"
  
  scp kube-proxy.kubeconfig \
    root@$host:/var/lib/kube-proxy/kubeconfig \
  
  scp admin.kubeconfig \
    root@$host:/var/lib/kubelet/kubeconfig
done
```

> [!TIP]
> POD_SUBNET for the controller node must be defined first in machines.txt. (unset by default)

Prepare KubeletConfiguration and copy it to controller instance:

```bash
for host in server; do
  SUBNET=$(grep $host machines.txt | cut -d " " -f 4)
    
  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml
    
  scp kubelet-config.yaml \
  root@$host:~/
done
```

Copy Kubernetes binaries and systemd unit files to controller instance:

```bash
for host in server; do
  scp \
    downloads/runc.arm64 \
    downloads/crictl-v1.28.0-linux-arm.tar.gz \
    downloads/cni-plugins-linux-arm64-v1.3.0.tgz \
    downloads/containerd-1.7.8-linux-arm64.tar.gz \
    downloads/kubectl \
    downloads/kubelet \
    downloads/kube-proxy \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kubelet-config.yaml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@$host:~/
done
```

Login to the controller instance using the `ssh` command. Example:

```bash
ssh root@server
```

## Provisioning worker components

Install the OS dependencies:

```bash
{
  apt-get update
  apt-get -y install socat conntrack ipset
}
```

Create the installation directories:

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```bash
{
  mkdir -p containerd
  tar -xvf crictl-v1.28.0-linux-arm.tar.gz
  tar -xvf containerd-1.7.8-linux-arm64.tar.gz -C containerd
  tar -xvf cni-plugins-linux-arm64-v1.3.0.tgz -C /opt/cni/bin/
  mv runc.arm64 runc
  chmod +x crictl kubectl kube-proxy kubelet runc 
  mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  mv containerd/bin/* /bin/
}
```

### Configure containerd

Install the `containerd` configuration files:

```bash
{
  mkdir -p /etc/containerd/
  mv containerd-config.toml /etc/containerd/config.toml
  mv containerd.service /etc/systemd/system/
}
```

### Configure the Kubelet

Create the `kubelet-config.yaml` configuration file:

```bash
{
  mv kubelet-config.yaml /var/lib/kubelet/
  mv kubelet.service /etc/systemd/system/
}
```

### Configure the Kubernetes Proxy

```bash
{
  mv kube-proxy-config.yaml /var/lib/kube-proxy/
  mv kube-proxy.service /etc/systemd/system/
}
```

### Start the Worker Services

```bash
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```

## Label and Taint the Controller Node

The node should be brought up at this point. To prevent regular workloads from being scheduled on the controller node, label it as a control plane node and then taint it:

```bash
kubectl label node server node-role.kubernetes.io/control-plane=
kubectl taint nodes server node-role.kubernetes.io/control-plane=:NoSchedule
```

This ensures that only pods specifically designated for the control plane will be scheduled on the controller node. 

**Note:** Replace `server` with your controller node's name if needed.