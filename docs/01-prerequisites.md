# Prerequisites

## OpenStack

This tutorial leverages [OpenStack][openstack] to handle provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up.

This repo is forked from Kelsey Hightower's original [kubernetes-the-hard-way][kelseyhightower/kubernetes-the-hard-way] repo which leverages Google Compute Engine (GCE) instead of OpenStack.

## Development Machine

This tutorial assumes your local machine runs Ubuntu Server 18.04 LTS amd64. You can spin such a machine up yourself using [Vagrant][vagrant].

On macOS, you might run:

```bash
mkdir -p ~/vagrant/dev
cd !$
vagrant init bento/ubuntu-16.04
vagrant up
vagrant ssh
```

## Download your OpenStack RC v3 file.

In the OpenStack dashboard, download the [OpenStack RC file][openstack-rc] for your OpenStack project. Save it as filename `openrc.sh` in the same directory where you ran `vagrant up` (e.g. `~/vagrant/dev/openrc.sh`). This directory is usually shared with the Vagrant machine at `/vagrant`. Putting the file here should make it readable from inside the Vagrant machine.

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)

<!--- Hidden References -->

[kelseyhightower/kubernetes-the-hard-way]: https://github.com/kelseyhightower/kubernetes-the-hard-way
[openstack-rc]: https://docs.openstack.org/ocata/user-guide/common/cli-set-environment-variables-using-openstack-rc.html
[openstack]: https://www.openstack.org
[vagrant]: https://www.vagrantup.com
