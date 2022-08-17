# Kubernetes on CoreOS on Premise Deployment

First you will need to aquire the CoreOS ISO at the time of writing I am using coreOS 36

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

```sudo rpm-ostree install kubelet kubeadm kubectl cri-o``` 

Additionally you can add `open-vm-tools` to that command if you're running in a VMware environement like myself. Due to the way CoreOS works after installing the packages you will have to reboot the server, you can use `sudo systemctl reboot` to do so. Once the server is back up you will need to enable the services:

```sudo systemctl enable --now crio kubelet```

Before we can initialize our cluster we will have to take some considerations on how to run our cluster in HA. Due to use being in an on premises environement we do not have a cloud load balancer meaning we will have to have our own. Kubernetes has some documentation on [their website](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) and [github](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing) covering this, I will personally be going with the keepalived and haproxy as static pods route. Follow their documentation on github for the specific one you want to use.

In my configuration I will be using port 6443 on the load balancer for the API and port 6444 as the API port on the masters. Once you have completed to configuration files for your load balancer route you're ready to initialise the cluster for this we will need a cluster config which I have added to this github repo. You then deploy the config using:
