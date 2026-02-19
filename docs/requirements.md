# Requirements

## Ansible

The requirements for all Ansible playbooks are listed in [/meta/requirements.yaml](../ansible/meta/requirements.yaml). Install them using `ansible-galaxy`:

    ansible-galaxy install -r ansible/meta/requirements.yaml

### Secrets Automation with 1Password

## Proxmox

Since I use Ansible [to clone my VMs](../ansible/swarm/bootstrap/create-vms.yaml), we need to provide a VM template in Proxmox VE.

## NFS

Since I also have state in my containers, I need to store the data in a way that is accesible on every node. This ensures that containers/services can be scheduled on every node.
For this I use a mounted NFS folder which is hosted in a simple LXC container on Proxmox VE.

The container is a simple Debian 13 LXC (see [Proxmox Docs](https://pve.proxmox.com/wiki/Linux_Container) for container template install guide) with the following settings:

- priviliged container
- nesting enabled
- NFS feature active
- start on boot
- large enough disk (64GB for me)
- 2 cores / 2048MB seems plenty

Then just install `nfs-kernel-server` in the container and create a share. I chose `/srv/nfs/swarm-data` for my share.

Configure export config with `nano /etc/exports`:

    /srv/nfs/swarm-data \
    192.168.20.11(rw,sync,no_subtree_check,no_root_squash) \
    192.168.20.12(rw,sync,no_subtree_check,no_root_squash) \
    192.168.20.13(rw,sync,no_subtree_check,no_root_squash)

Notice that we only allow the 3 swarm nodes access.

Restart:

    sudo exportfs -a
    sudo systemctl restart nfs-kernel-server

The requirements for each node are handled by [this Ansible playbook](../ansible/swarm/bootstrap/connect-nfs.yaml).
