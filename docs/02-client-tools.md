# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial:

- [cfssl][cfssl]
- [cfssljson][cfssl]
- [jq][jq]
- [kubectl][kubectl]
- [python-neutronclient][openstack-clients]
- [python-openopenstack-client][openstack-clients]

## Install OpenStack Clients.

This tutorial uses common OpenStack clients to interact with OpenStack.

You can usually install the necessary OpenStack clients on your dev machine like this:

```
sudo apt install -y python-dev python-pip
sudo pip install python-openstackclient python-neutronclient
```

## Install jq.

[jq] is used to parse JSON.

```
sudo apt install -y jq
```

## Install CFSSL.

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Download and install `cfssl` and `cfssljson` from the [cfssl repository](https://pkg.cfssl.org):

```bash
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### Verification

Verify `cfssl` version 1.2.0 or higher is installed:

```bash
cfssl version
# Version: 1.2.0
# Revision: dev
# Runtime: go1.6
```

> The cfssljson command line utility does not provide a way to print its version.

## Install kubectl.

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.12.4/bin/linux/amd64/kubectl

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Verification

Verify `kubectl` version 1.12.4 or higher is installed:

```bash
kubectl version --client
# Client Version: version.Info{Major:"1", Minor:"12", GitVersion:"v1.12.4", GitCommit:"0ed33881dc4355495f623c6f22e7dd0b7632b7c0", GitTreeState:"clean", BuildDate:"2018-09-27T17:05:32Z", GoVersion:"go1.10.4", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning Compute Resources](03-compute-resources.md)

<!--- Hidden References -->

[cfssl]: https://github.com/cloudflare/cfssl
[jq]: https://stedolan.github.io/jq/
[kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl
[openstack-clients]: https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html
