# Setup Instructions

## Set up your domain name

In your domain registry, set the root domain (`@`) and the `nextcloud.` subdomaing to point to your public IP address.

e.g. `<your>.<domain>` & `nextcloud.<your>.<domain>` should point to your public ip address.

## Setup PI and configure router

1. Find Pi ip address (Just wait for DHCP). Can be done via looking in the DHCP table of the router or funning `arp -na | grep -i "dc:a6:32"` (PI4)
2. Set the hostname to something useful, i.e jonopi4
    ```bash
    $ sudo nvim /etc/hostname
    $ sudo sync
    $ sudo reboot
    ```
3. Restart the router (so the new hostname gets applied)
4. In the the configuration for the home router, set a static IP address bassed on the mac address of the Raspberry Pi

## Install Microk8s

Install as per the docs. 

Once Microk8s is up and running make sure to check the status of the nodes. Note the default node is the hostname

```bash
$ kubectly get nodes

NAME     STATUS   ROLES    AGE     VERSION
jonopi4   Ready    <none>   7d22h   v1.18.6-1+b4f4cb0b7fe3c1
```

If the default ubunto node is not ready, then follow the addivce here to fix the issue.

[enable cgroup memory](https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap/1190457#1190457)

## Install HELM and add official Repo

Install [Helm](https://helm.sh) as per Website.

Make sure to add the official repo and the nextcloud repo.

```bash
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

# Mount SSD drive

```bash
// determine what the ssd is.
$ fdisk -l

// assume dist is sda1
$ mkfs.ext4 /dev/sda1

// mount the drive
$ mkdir /mnt/ssd
$ chown -R ubuntu:ubuntu /mnt/ssd/
$ mount /dev/sda1 /mnt/ssd

// automatically mount the drive on startup
// get the uuid
$ sudo blkid

// edit the fstab and enter the below
$ sudo nvim /etc/fstab
UUID=88a2b5fb-4a03-4346-b3d4-9ba818820f93 /mnt/ssd ext4 defaults 0 0
```

Sycn and reboot. Confirm the disk works

```bash
$ df -ha /dev/sda1
```

# Optional - If you want to share the drives accross multiple nodes

On master server
```bash
// install all dependencies
$ sudo apt-get install nfs-kernel-server -y

// add to the exports
$ sudo nvim /etc/exports
/mnt/ssd *(rw,no_root_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)

// start the server
$ sudo exportfs -ra
```

On workers

```bash
// install dependecies
$ sudo apt-get install nfs-common -y

// create the directory
$ sudo mkdir /mnt/ssd
$ sudo chown -R ubuntu:ununtu /mnt/ssd/

// configure auto mount <master ip address>
$ sudo nvim /etc/fstab
$ 192.168.1.100:/mnt/ssd   /mnt/ssd   nfs    rw  0  0
```

Sync, reboot.

## Setup Traefik revers proxy.

Edit the `02.02.traefik.yml` file and change your email address and host ip address

Apply the Kubernetes resources

```bash
$ kubectl apply -f 02.01.traefik.yml`
$ kubectl apply -f 02.02.traefik.yml`
```

The Traefik dashboard should now be accessible on port `:8100`

## Install NextCloud

helm show values nextcloud/nextcloud >> nextcloud.values.yml


Add the [nextcloud](https://nextcloud.github.io/helm) repo.

```
$ helm repo add nextcloud https://nextcloud.github.io/helm/
```

Copy the default value to file.

```
$ helm show values nextcloud/nextcloud >> 03.02.nextcloud.values.yml
```

Edit the `03.02.nextcloud.value.yml` and change the following

```yaml
nextcloud:
  host: nextcloud.<your>.<domain>
  username: <USERNAME>
  password: <PASSWORD>
  ...

  # Set so that redirects use 'https://' protocal'
  extraEnv:
    - name: NC_overwriteprotocol
      value: https
...

persistence:
  enabled: true # Change to true
  existingClaim: "nextcloud-ssd" # Persistent Volume Claim created earlier
  accessMode: ReadWriteOnce
  size: "50Gi"
```

Create the NextCloud Deployment

```bash
$ helm install nextcloud --namespace nextcloud -f 03.02.nextcloud.values.yml nextcloud/nextcloud
```

Create the ingress route for your NextCloud Deployoment.

Edit the host name to match your domain in `03.03.nextcloud.ingress.yml` then apply the ingress route.

```
$ kubectl apply -f 03.03.nextcloud.ingress.yml`
```

Once the deployment is up and running, the Traefik revers proxy should detect the IngressRoute for the `nextcloud.<your>.<domain>`
and auto create the SSL certificates on demand.

Your `NextCloud` service should now be available at `https://nextcloud.<your>.<domain>`

# Setup the Pi-Hole on Kubertenetes

Edit the `04.pi-hole.yml` with the correct PI4 local ip address and admin secret

Apply the pi-hole

```bash
$ kubectl apply -f 04.pi-hole.yml
```

[comment]: # Ignore below for now
[comment]: # Create directories for the persistant data. Two located in /usr/home/dat
[comment]: # `/home/ubuntu/data/pi-hole` & `/home/ubuntu/data/dnsmasq`


Disable the DNS on the router and set to satic ip of raspbery pi, restart the router

