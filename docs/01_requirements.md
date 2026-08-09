# Requirements for bootstraping the cluster

## Ansible

The requirements for all Ansible playbooks are listed in [/meta/requirements.yaml](../ansible/meta/requirements.yaml). Install them using `ansible-galaxy`:

    ansible-galaxy install -r ansible/meta/requirements.yaml

### Secrets Automation with 1Password

## Proxmox

### VM Template

Since I use Ansible [to clone my VMs](../ansible/swarm/bootstrap/create-vms.yaml), we need to provide a VM template in Proxmox VE.

My VM template is based on [this guide](https://technotim.com/posts/cloud-init-cloud-image/) by Techno Tim.

As an image I currently use to newest version (as of August 2026, daily 20260731) of Ubuntu 26.04 LTS Resolute Raccoon.

Therefor the template is created with these commands (run in root shell of Proxmox host):

    # download image
    wget https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img

    # create VM with 2 Cores / 2GB (not relevant, later changed with Ansible)
    qm create 9000 --memory 2048 --core 2 --name ubuntu-cloud-ansible --net0 virtio,bridge=vmbr0
    # import disk into vm
    qm disk import 9000 resolute-server-cloudimg-amd64.img local-lvm
    qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
    # add cloud init drive
    qm set 9000 --ide2 local-lvm:cloudinit
    # boot options
    qm set 9000 --boot c --bootdisk scsi0
    # add serial
    qm set 9000 --serial0 socket --vga serial0

    # convert to template
    qm template 9000

Now prepare the template

- resize disk in "Hardware", once by 0.5 and once by 12
- in "Cloud-Init":

        User: serveradmin
        Password: S3cure # please change me
        SSH public key: ssh-ed25519 # please change me

- hit "Regenerate Image"
- remove the Cloud-Init drive in "Hardware"
- in "Options":

        Start at Boot: yes
        QEMU Guest Agent: Enabled

### NFS CT

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
    10.14.20.11(rw,sync,no_subtree_check,no_root_squash) \
    10.14.20.12(rw,sync,no_subtree_check,no_root_squash) \
    10.14.20.13(rw,sync,no_subtree_check,no_root_squash)

Notice that we only allow the 3 swarm nodes access.

Restart:

    sudo exportfs -a
    sudo systemctl restart nfs-kernel-server

The requirements for each node are handled by [this Ansible playbook](../ansible/swarm/bootstrap/connect-nfs.yaml).

## Start CI/CD pipeline with doco-cd

This repository uses [doco-cd](https://github.com/kimdre/doco-cd) for CD. Unlike other tools (like Flux for k8s) doco-cd can't mange itself (yet). This means we need to boostrap the cluster by starting doco-cd. This is done using the Docker Compose file in the root of the repository. I am running it in swarm mode, so I need to deploy doco-cd as a stack:

    docker stack deploy -c docker-compose.yaml -d doco-cd

I do this on `swarm01` after copying over the compose file with SCP:

    scp docker-compose.yaml serveradmin@192.168.20.11:/home/serveradmin/docker-compose.yaml

It`s not fully automated (yet), but good enough.
