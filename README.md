# Kubernetes on CoreOS on Premise Deployment

First you will need to aquire the CoreOS ISO at the time of writing I am using coreOS 36

[CoreOS Download URL](https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable&arch=x86_64)

To deploy the CoreOS you will need something call an ignition config, Once you boot the ISO in your virtual machine it will request a ignition file or an ignition URL. I have added a base ignition config however, it relies on DHCP to provide the hostname and IP address if you want to apply it statically add the code block below to storage, files.

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
