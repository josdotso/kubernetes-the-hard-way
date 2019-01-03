# Deploying the DNS Cluster Add-on

In this lab you will deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery, backed by [CoreDNS](https://coredns.io/), to applications running inside the Kubernetes cluster.

## Configure instance-to-instance DNS resolution

OpenStack, unlike Google Cloud, does not usually enable automatic private subnet DNS resolution. This means that instances cannot look up one another in internal DNS in OpenStack they way they can in GCE.

To mitigate this, you can generate a custom `/etc/hosts` file locally and upload it to your OpenStack instances.

Generate the `hosts` file.

```bash
{
  cat << EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
EOF
  echo
  openstack server list --format value | awk -F'[ ,=]' '{ print $5, $2 }'
} | tee hosts
# 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
# ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
# 
# 10.240.0.5 bastion-0
# 10.240.0.22 worker-2
# 10.240.0.21 worker-1
# 10.240.0.20 worker-0
# 10.240.0.12 controller-2
# 10.240.0.11 controller-1
# 10.240.0.10 controller-0
```

Distribute and install the `hosts` file on all instances.

```bash
for instance in bastion-0 \
    controller-0 controller-1 controller-2 \
    worker-0 worker-1 worker-2
do
  ssh ${instance} "sudo cp /etc/hosts /etc/hosts.backup.$RANDOM"
  ssh ${instance} "sudo tee /etc/hosts" < hosts
done
```

## The DNS Cluster Add-on

Deploy the `coredns` cluster add-on:

```bash
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
# serviceaccount/coredns created
# clusterrole.rbac.authorization.k8s.io/system:coredns created
# clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
# configmap/coredns created
# deployment.extensions/coredns created
# service/kube-dns created
```

List the pods created by the `kube-dns` deployment:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-699f8ddd77-94qv9   1/1     Running   0          20s
# coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

## Verification

Create a `busybox` deployment:

```bash
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

List the pod created by the `busybox` deployment:

```bash
kubectl get pods -l run=busybox
# NAME                      READY   STATUS    RESTARTS   AGE
# busybox-bd8fb7cbd-vflm9   1/1     Running   0          10s
```

Retrieve the full name of the `busybox` pod:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
# Server:    10.32.0.10
# Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local
# 
# Name:      kubernetes
# Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [Smoke Test](13-smoke-test.md)
