# Kubernetes on CoreOS on Premise Deployment

Update! - I wrote this guide around 2 years ago, and I am not sure wether it's still functional. Anyways, I would highly recommend to check out [Talos Linux](https://www.talos.dev/) it will pretty much set up the cluster for you and will probably be a more reliable solution then self administrating this on CoreOS.

I am no pro at writing guides and this is honestly my first one, I will be installing everything needed to have a proper "cloud like" cluster at home if I have made any mistakes please tell me. First you will need to aquire the CoreOS ISO at the time of writing I am using coreOS 36

[CoreOS Download URL](https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable&arch=x86_64)

To deploy the CoreOS you will need something called an [ignition config](https://github.com/nyanmark/coreos-k8s/blob/main/ignition.yaml) this will configure your new VM. For more information on ignition you can look in the [fedora docs](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/). Once you boot the ISO in your virtual machine it will request a ignition file or an ignition URL. I have added a base ignition config however, it relies on DHCP to provide the hostname and IP address if you want to apply it statically add the code block below to storage, files.

```
    - path: /etc/hostname
      mode: 420
      contents:
        source: "data:,<YOUR HOSTNAME>"
    - path: /etc/NetworkManager/system-connections/enp1s0.nmconnection
      mode: 0600
      overwrite: true
      contents:
        inline: |
          [connection]
          type=ethernet
          id="Wired connection 1"
          interface-name=<INTERFACE NAME>

          [ipv4]
          method=manual
          addresses=<YOUR IP/MASK>
          gateway=<YOUR GW>
          dns=<YOUR DNS>
```

Additionally if you do not wish to use an SSH key or want to have a password just incase you can add `password_hash: <YOUR ENCODED PASSWORD>` to the users definition. The password has to be encorded to do so use the `mkpasswd --method=yescrypt` command from the whois package.

Once you have your YAML file ready you will have to encode it into a JSON format this is done through a program called butane which you can [download here](https://github.com/coreos/butane/releases). I personally use linux so I download the latest version and place it into my `/usr/local/bin/butane` don't forget to `chmod +x /usr/local/bin/butane` to make it executable. Now we can create our .ign file with the command:

```butane --pretty --strict ignition.yaml > config.ign```

I will then serve those files using `python3 -m http.server 80` from my desktop and deploy the CoreOS servers. Finally we will be able to SSH into our CoreOS nodes. I will be using CRI-O as my Container Runtime. To install the required packages you will need to execute:

```sudo rpm-ostree install kubelet kubeadm kubectl cri-o open-vm-tools``` 

If you stumble upon this error `error: failed to parse public key for /var/cache/rpm-ostree/repomd/kubernetes-36-x86_64/yum-key.gpg` just run `rm /var/cache/rpm-ostree/repomd/kubernetes-36-x86_64/yum-key.gpg` and re run the rpm command. Due to the way CoreOS works after installing the packages you will have to reboot the server, you can use `sudo systemctl reboot` to do so. Once the server is back up you will need to enable the services:

```sudo systemctl enable --now crio kubelet```

Before we can initialize our cluster we will have to take some considerations on how to run our cluster in HA. Due to use being in an on premises environement we do not have a cloud load balancer meaning we will have to have our own. Kubernetes has some documentation on [their website](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) and [github](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing) covering this, I will personally be going with the keepalived and haproxy as static pods route. Follow their documentation on github for the specific one you want to use.

In my configuration I will be using port 6443 on the load balancer for the API and port 6444 as the API port on the masters. The keepalived file can cause problems if the permissions on it are wrong, I got it working with `chown root:root` and `chmod 0644`. Once you have completed to configuration files for your load balancer route you're ready to initialise the cluster for this we will need a cluster config which I have added to this github repo. You then deploy the config using:

```kubeadm init --upload-certs --config=cluster-config.yaml```

Once your first master is deployed save the join command displayed to you needed to deploy the other master nodes. It is annoying to recover if lost. The next step would be to install your network plugin, in my case I will be using [calico](https://www.tigera.io/project-calico/) as of writing this I am using CoreOS 36, Kubernetes 1.24.3 and Calico 3.24. To install calico:

```kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.0/manifests/tigera-operator.yaml```

Then create a file for the calico custom resource, if you're following their guide all you have to edit is the IPv4 block and add the `flexVolumePath` variable for CoreOS as the default path used by calico is unwritable. Also feel free to install the calicoctl software on your host:

```curl -L https://github.com/projectcalico/calico/releases/download/v3.24.0/calicoctl-linux-amd64 -o /usr/local/bin/calicoctl```

```chmod +x /usr/local/bin/calicoctl```

```
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  flexVolumePath: /etc/kubernetes/kubelet-plugins/volume/exec
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
```

```kubectl create -f custom-resource.yaml```

To verify calico's status run `kubectl get pods -n calico-system` all the pods should be running. This means you can add your other control plane nodes to the cluster using the join command you saved earlier add `--apiserver-bind-port <YOUR PORT>` to the end of the join command so the API server starts on the correct port in my case the port is 6444. You can run `kubectl get nodes` to see if they have joined successfully:

```
NAME      STATUS   ROLES           AGE   VERSION
master0   Ready    control-plane   14m   v1.24.3
master1   Ready    control-plane   86s   v1.24.3
master2   Ready    control-plane   16s   v1.24.3
```

We can finally start working on the worker nodes in our cluster for the sake of this I will also be doing 3 CoreOS worker nodes. To deploy my worker virtual machines I will be using the same ignition configs as before we will also need the iscsi package wbich is pre-installed on coreos this is needed for the storage backend we are going to use [OpenEBS](https://openebs.io/) cStor, this will let us have HA distributed storage on site, [Rook](https://rook.io/) Ceph also does this well. Once the packages have been installed and the server rebooted we can get ready to add it to our cluster first some commands will need to be ran to prepare it for OpenEBS after this the join command can be ran to add the server to the cluster.

```sudo systemctl enable --now crio kubelet iscsid```

```modprobe iscsi_tcp```

```echo iscsi_tcp >/etc/modules-load.d/iscsi-tcp.conf```

```kubeadm join <DATA>```

Once the servers have joined the cluster you can confirm this with a `kubectl get nodes` which should output a list similar to the one below. The next step would be configuring the storage I will be using [this guide](https://github.com/openebs/cstor-operators/blob/develop/docs/quick.md) from OpenEBS to deploy my storage and I will need to add an aditional disk to each worker node.

```
NAME      STATUS   ROLES           AGE     VERSION
master0   Ready    control-plane   153m    v1.24.3
master1   Ready    control-plane   140m    v1.24.3
master2   Ready    control-plane   138m    v1.24.4
worker0   Ready    <none>          4m33s   v1.24.4
worker1   Ready    <none>          3m44s   v1.24.4
worker2   Ready    <none>          96s     v1.24.4
```

After we got our storage ready we need to configure a couple more things which is the load balancer and reverse proxy, as we do not have a cloud load balancer we will be using a project called [metallb](https://metallb.io) which will allow us to run a load balancer right inside k8s. You will have to chose the mode either ARP or BGP the installation guide is [located here](https://metallb.universe.tf/installation/) if you want to run BGP you will have to be aware of calico also running its own bgp service meaning you will have to use the patch [from here](https://metallb.universe.tf/configuration/calico/).

For the reverse proxy I will be using the default [nginx ingress](https://github.com/kubernetes/ingress-nginx) the installation guide is [here](https://kubernetes.github.io/ingress-nginx/deploy/) once you got this deployed you should have a fully functional cluster just like those in the cloud with HA storage and load balancers. If you want to get some logging going in your cluster I would recommend following [this guide](https://medium.com/nerd-for-tech/logging-at-scale-in-kubernetes-using-grafana-loki-3bb2eb0c0872) however, remember to set your sc as the default for the helm to work properly and due to coreos security the promtail container will not start unless you edit the daemonset/loki-promtail (kubectl edit) and set allow privilige escalation and privilieged to true.
