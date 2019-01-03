# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

The commands in this lab must be run on each controller instance: `controller-0`, `controller-1`, and `controller-2`. Login to each controller instance using the `ssh` command. Example:

```bash
ssh controller-0
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```bash
sudo mkdir -p /etc/kubernetes/config
```

### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries:

```bash
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.4/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.4/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.4/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.4/bin/linux/amd64/kubectl"
```

Install the Kubernetes binaries:

```bash
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Configure the Kubernetes API Server

```bash
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:

```bash
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

Create the `kube-apiserver.service` systemd unit file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place:

```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into place:

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the `kube-scheduler.yaml` configuration file:

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Create the `kube-scheduler.service` systemd unit file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Controller Services

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

### Enable HTTP Health Checks

In some OpenStack clouds it is possible configure a cloud load balancer to health check on a different port from the port being load balanced. This section is applicable in only those OpenStack clouds. And then, this section is only usually applicable when your Kubernetes API certificates are self-signed, which is usually the case.

OpenStack's load balancing service may include the ability to health check HTTPS endpoints. If your OpenStack load balancer service does not, then this section is probably applicable.

As an alternative to configuring HTTP-based health checks, you may also choose to health check Kubernetes API using TCP health checks.

As a workaround for the HTTPS and self-signed issues, the nginx webserver can be used to proxy HTTP health checks to HTTPS endpoints. In this section nginx will be installed and configured to accept HTTP health checks on port `80` and proxy the connections to the API server on `https://127.0.0.1:6443/healthz`.

> The `/healthz` API server endpoint does not require authentication by default.

Install a basic web server to handle HTTP health checks:

```bash
sudo apt-get install -y nginx
```

```bash
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

```bash
{
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
}
```

```bash
sudo systemctl restart nginx
```

```bash
sudo systemctl enable nginx
```

### Verification

```bash
kubectl get componentstatuses --kubeconfig admin.kubeconfig
# NAME                 STATUS    MESSAGE              ERROR
# controller-manager   Healthy   ok
# scheduler            Healthy   ok
# etcd-2               Healthy   {"health": "true"}
# etcd-0               Healthy   {"health": "true"}
# etcd-1               Healthy   {"health": "true"}
```

Test the nginx HTTP health check proxy:

```bash
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
# HTTP/1.1 200 OK
# Server: nginx/1.14.0 (Ubuntu)
# Date: Sun, 30 Sep 2018 17:44:24 GMT
# Content-Type: text/plain; charset=utf-8
# Content-Length: 2
# Connection: keep-alive
# 
# ok
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.


## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.

```bash
ssh controller-0
```

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

The Kubernetes API Server authenticates to the Kubelet as the `kubernetes` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kubernetes` user:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## The Kubernetes Frontend Load Balancer

A [Load Balancer as a Service (LBaaS)][lbaas] v2 load balancer will be used to distribute traffic across the three API servers and allow each API server to terminate TLS connections and validate client certificates.

In this section you will provision an external load balancer to front the Kubernetes API Servers. The floating IP address you created earlier for Kubernetes API Server will be attached to the resulting load balancer.

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.


### Provision a LBaaS v2 Load Balancer

Create the external load balancer resources:

```bash
{
  KUBERNETES_PUBLIC_ADDRESS=X.X.X.X  # the address of the floating IP you created Kubernetes API earlier.

  neutron lbaas-loadbalancer-create --name kubernetes-the-hard-way kubernetes

  lb_vip_port_id=$(neutron lbaas-loadbalancer-show kubernetes-the-hard-way --format json | jq -r .vip_port_id)

  neutron port-update --security-group kubernetes-the-hard-way-allow-external ${lb_vip_port_id}

  neutron lbaas-listener-create --name kubernetes \
    --loadbalancer kubernetes-the-hard-way --protocol HTTPS --protocol-port 6443

  neutron lbaas-pool-create --name kubernetes-target-pool \
    --lb-algorithm ROUND_ROBIN --listener kubernetes --protocol HTTPS

  neutron lbaas-member-create --subnet kubernetes --address 10.240.0.10 \
    --protocol-port 6443 kubernetes-target-pool --name controller-0

  neutron lbaas-member-create --subnet kubernetes --address 10.240.0.11 \
    --protocol-port 6443 kubernetes-target-pool --name controller-1

  neutron lbaas-member-create --subnet kubernetes --address 10.240.0.12 \
    --protocol-port 6443 kubernetes-target-pool --name controller-2

  neutron lbaas-healthmonitor-create --delay 5 --max-retries 2 --timeout 10 \
    --type HTTPS --pool kubernetes-target-pool --url-path /healthz \
    --name kubernetes-api-healthmonitor

  floatingip_id=$(openstack floating ip show ${KUBERNETES_PUBLIC_ADDRESS} --format json | jq -r .id)

  neutron floatingip-associate ${floatingip_id} ${lb_vip_port_id}
}
```

Associate the floating IP:

```bash
{
  KUBERNETES_PUBLIC_ADDRESS=X.X.X.X  # the address of the floating IP you created Kubernetes API earlier.

  neutron floatingip-associate FLOATINGIP_ID LOAD_BALANCER_PORT_ID
}
```

### Verification

Make an HTTP request for the Kubernetes version info:

```bash
KUBERNETES_PUBLIC_ADDRESS=X.X.X.X  # the address of the floating IP you created for Kubernetes API earlier.

curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> output

```json
{
  "major": "1",
  "minor": "12",
  "gitVersion": "v1.12.4",
  "gitCommit": "0ed33881dc4355495f623c6f22e7dd0b7632b7c0",
  "gitTreeState": "clean",
  "buildDate": "2018-09-27T16:55:41Z",
  "goVersion": "go1.10.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

**NOTE:** Due to the self-signed / HTTPS health-checking issues discussed above, you may see annoying but mostly harmless messages like `http: TLS handshake error from 10.240.0.15:43522: tls: client offered an unsupported, maximum protocol version of 300` in your `kube-apiserver.service` logs. These are generally safe to ignore. See: [issue #70411][issue-70411]

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)

<!--- Hidden References -->

[issue-43784]: https://github.com/kubernetes/kubernetes/issues/43784
[issue-70411]: https://github.com/kubernetes/kubernetes/issues/70411
[lbaas]: https://docs.openstack.org/mitaka/networking-guide/config-lbaas.html
