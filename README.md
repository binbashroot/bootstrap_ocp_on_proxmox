Deploy OCP on Proxmox
=========

A repository for hands off provisioning OCP on Proxmox. Currently only supports "legacy bios" provisioning.  
Keep in mind this is for an internal home lab setting. YMMV
Installation of PXE booting, DHCP, Ansible, or Nginx is not covered in this document

**Note:**   
*For this lab deployment there is shared storage between the nginx, tftp, and ansible resources.  This is to simplify the deployment.  The containers use bind volume mounts to their respective containter mount points while the Ansible server uses NFS.*

Requirements
------------
##### Services
| Service | Type | Notes |
|:---|:---|:---|
| ansible | Virtual Machine | House deployment playbooks** |
| dhcp | Router | Provide `static` dhcp addresses to OCP virtual machines for pxe booting |
| tftp | Podman container | Holds the boot.ipxe and undionly.kpxe files |
| nginx | Podman container | Contains ipxe, ignition, and rhcos img files |


##### Software
| Name | Version | Notes |
|:---|:---|:---|
| Ansible | 2.18.12 | You can probably use 2.16 or or higher, but all my testing was with 2.18.12 |
| Python | 3.14.2 | You can probably use 3.11+, but my testing was with 3.14.2 |
| Jinja | 3.1.6 | - |
| openshift-install | 4.21.0 | Can be downloaded from RH or the GH repository for the installer |
| ipxe-bootimgs | 20200823 | Use: `dnf install ipxe-bootimgs-x86` to install. Contains the undionly.kpxe binary which you need on your tftp server|

DHCP
------------
It's recommended to use static DHCP entries on your dhcp server.  How to set up a dhcp is not covered in this document.  Only the settings required for your DHCP server to properly serve tftp are covered.
Do yourself a favor and configure your virtual machines with static mac addresses, and assign them a static dhcp address in your router or dhcp software.
| Code | Name | Value | Notes |
|:---|:---|:---|:---|
| 66 | tftp-server | IP of tftp server |
| 67 | tftp-boot-file | boot.ipxe | Contains mac and web server information for provisioned host |

TFTP
------------
Where you host the tftp files doesn't matter.  What really matters is that you configure your tftp server to redirect to your web server.  Doing downloads from a a web server is faster than using TFTP itself because TFTP is very slow. Using a docker container via a NAS is perfect for this as it's small and uses hardly any resources. It is also low overhead and can be stopped when not needed. Configuration of tftp is not covered in this document
| File | Notes |
|:---|:---|
| boot.ipxe | Contains webserver url and ignition file to use when booting OCP host |
| undionly.pkxe | IPXE boot loader for legacy bios. Acts as bridge allowing ipxe to use http or other protocols |

#### boot.ipxe 
````shell
#!ipxe

echo Initializing network...
dhcp net0
set web-url http://10.10.10.100:7880
echo Booting node with MAC: ${net0/mac}
# MAC ADDRESSES OF bootstrap, master, and worker nodes
iseq ${net0/mac} bc:24:11:12:34:56 && chain ${web-url}/bootstrap.ipxe ||
iseq ${net0/mac} bc:24:11:23:45:67 && chain ${web-url}/master.ipxe ||
iseq ${net0/mac} bc:24:11:45:67:89 && chain ${web-url}/master.ipxe ||
iseq ${net0/mac} bc:24:11:01:23:45 && chain ${web-url}/master.ipxe ||
iseq ${net0/mac} bc:24:11:67:89:10 && chain ${web-url}/worker.ipxe ||
iseq ${net0/mac} bc:24:11:22:34:56 && chain ${web-url}/worker.ipxe ||
iseq ${net0/mac} bc:24:11:78:90:10 && chain ${web-url}/worker.ipxe ||

# Fallback to shell if MAC doesn't match
echo MAC ${net0/mac} not recognized.
shell
````

Ngnix
------------
- The web server uses both http and https protoocol.
- Will host your OCP ignition files as well as the OCP .img/.iso files for bootstrapping your OCP hosts. 

***Note:***
*Be sure to configure the web server to only allow specific OCP hosts/subnets to access the bootstrap directory.*

##### Required files in Web Server
| File | Usage | Persistent | Notes |
|:---|:---|:---|:---|
| bootstrap.ign | Bootstrap node ignition file | no | Generated during playbook run |
| master.ign | Control node ignition file | no | Generated during playbook run |
| worker.ign | Worker node ignition file | no | Generated during playbook run |
| bootstrap.ipxe | Bootstrap ipxe file | yes | Manually creat once. Points to ignition files|
| master.ipxe | Control ipxe file | yes | Manually create once. Points to ignition files |
| worker.ipxe | Worker ipxe file | yes | Manually created once. Points to ignition files |
| rhcos-live-kernel.x86_64 | Used for OCP installation |  yes | Download once |
| rhcos-live-iso.x86_64.iso | Used for OCP installation |  yes | Download once |
| rhcos-live-initramfs.x86_64.img | Used for OCP installation | yes | Download once |
| rhcos-live-rootfs.x86_64.img | Used for OCP installation | yes | Download once |


IPXE Files
--------------
The ipxe files control the pxe boot params for a host.
| File | Location | Usage | Notes |
|:---|:---|:---|:---|
| boot.ipxe | tftp server | Defines web server IP, identifies boot host by mac, and chainloads ipxe boot params |
| bootstrap.ipxe | web server | Configures bootstrap server to use bootstrap ignition file |
| master.ipxe | web server | Configures bootstrap server to use master ignition file |
| master.ipxe | web server | Configures bootstrap server to use worker ignition file |

#### bootstrap.ipxe
````shell
# EXAMPLE bootstrap.ipxe file
#!ipxe
set web-url http://10.10.10.100:7880

echo Loading RHCOS Kernel for Bootstrap Node...
kernel ${web-url}/rhcos-live-kernel.x86_64 \
    initrd=rhcos-live-initramfs.x86_64.img \
    coreos.live.rootfs_url=${web-url}/rhcos-live-rootfs.x86_64.img \
    coreos.inst.install_dev=/dev/sda \
    coreos.inst.ignition_url=${web-url}/bootstrap.ign \
    ip=dhcp \
    console=tty0 console=ttyS0,115200

echo Loading RHCOS Initrd...
initrd ${web-url}/rhcos-live-initramfs.x86_64.img

boot
````
#### worker.ipxe file
````shell
#!ipxe
set web-url http://10.10.10.100:7880

echo Loading RHCOS Kernel for Bootstrap Node...
kernel ${web-url}/rhcos-live-kernel.x86_64 \
    initrd=rhcos-live-initramfs.x86_64.img \
    coreos.live.rootfs_url=${web-url}/rhcos-live-rootfs.x86_64.img \
    coreos.inst.install_dev=/dev/sda \
    coreos.inst.ignition_url=${web-url}/worker.ign \
    ip=dhcp \
    console=tty0 console=ttyS0,115200

echo Loading RHCOS Initrd...
initrd ${web-url}/rhcos-live-initramfs.x86_64.img

boot
````

#### master.ipxe file
````shell
#!ipxe
set web-url http://10.10.10.100:7880

echo Loading RHCOS Kernel for Bootstrap Node...
kernel ${web-url}/rhcos-live-kernel.x86_64 \
    initrd=rhcos-live-initramfs.x86_64.img \
    coreos.live.rootfs_url=${web-url}/rhcos-live-rootfs.x86_64.img \
    coreos.inst.install_dev=/dev/sda \
    coreos.inst.ignition_url=${web-url}/master.ign \
    ip=dhcp \
    console=tty0 console=ttyS0,115200

echo Loading RHCOS Initrd...
initrd ${web-url}/rhcos-live-initramfs.x86_64.img

boot
````

Variables
--------------
When placing your variables, remember variable precendence.  Variables should be in your group_vars and host_vars within your inventory to enable consistent builds each time. Keep in mind that the bootstrap, control, and worker nodes all have different vm requirements so using groups is more efficient but host_vars allow for the granularity that may be required for your environment

#### Example host_vars

```yaml
# Example host var
# ocp-bootstrap.example.com
---

mac_address: 'AA:24:11:21:BB:50'
proxmox_node: proxmox4
vmid: 5007
```

#### Example group_vars
````yaml
# all.yml
proxmox_api_host: proxmox.example.com
proxmox_token_id: token1
proxmox_api_pass: **********
proxmox_token_secret: **********************
proxmox_api_user: svc_proxmox@EXAMPLE.COM

# bootstrap.yml
proxmox_aio: io_uring
proxmox_cores: 4
proxmox_disk_discard: on
proxmox_disk_format: raw
proxmox_disk_size: 120
proxmox_ram: 16384
proxmox_storage: local-lvm
proxmox_startup: 'order=1'
proxmox_tags:
  - ocp
  - bootstrap

# controlnodes.yml
proxmox_aio: io_uring
proxmox_cores: 4
proxmox_disk_discard: on
proxmox_disk_format: raw
proxmox_disk_size: 120
proxmox_ram: 20480
proxmox_storage: local-lvm
proxmox_startup: 'order=1'
proxmox_tags:
  - ocp
  - controlnode

# workernodes.yml
proxmox_aio: io_uring
proxmox_iothread: true
proxmox_cores: 16
proxmox_disk_discard: ignore
proxmox_disk_format: qcow2
proxmox_disk_size: 200
proxmox_numa: yes
proxmox_ram: 82944
proxmox_storage: proxmox_nvme
proxmox_tags:
  - ocp
  - workernode
proxmox_startup: 'order=2,up=60'
````

| Variable | Type | Default | Required | Notes |
|:------|:-|:-|:-|:--------|
| proxmox_api_host | string | - | yes | Proxmox API hostname/IP |
| proxmox_api_user | string | omit | yes | Proxmox API username |
| proxmox_api_pass | string | omit | no* | Not required if using tokenid/token secret |
| proxmox_token_id | string | omit | no* | Not required if using username/password |
| proxmox_token_secret | string | omit | no* | Not required if using username/password |
| proxmox_balloon | int | 0 | no | You don't want ballooning in OCP |
| proxmox_bios | string | seabios | no | Legacy Bios |
| proxmox_cores | int | 4 | yes* | Cores allocated to VM.  See group_vars |
| proxmox_cpu_type | string | host | no | Use host CPU for VM |
| proxmox_hotplug | string | network,cpu,memory,disk | no | Hot plug devices |
| proxmox_ram | int | 16384 | yes* | VM Memory - see group_vars |
| proxmox_vm_name | string | {{ inventory_hostname }} | yes | VM Name in Proxmox UI |
| proxmox_mac_address | string | - | yes* | See host_vars |
| proxmox_node | string | - | yes* | Node VM will be built on.  See host_vars |
| proxmox_numa | string | no | no* | Numa required for workernodes |
| proxmox_onboot | string | no | no | Playbook controls when host is booted |
| proxmox_ostype | string | l26 | no | Always use l26 |
| proxmox_protection | string | false | no | Not worth setting |
| promxox_scsihw | string | virtio-scsi-single | yes | Required for performance |
| proxmox_sockets | int | 1 | yes* | Typically 1 is enough see cores** |
| proxmox_startup | string | - | yes* | See group vars.  Workers must start after controlnodes |
| proxmox_storage | string | local-lvm | yes* | See group_vars.  Workers are on different storage and use qcow2 format |
| proxmox_tags | list | - | no | Tags in proxmox which are helpful when using a dynamic inventory | 
| proxmox_validate_certs | bool | true | no | set to false if not validating ssl certs |
| proxmox_vcpus | int | omit | no | If you want to use vcpus vs cores.  For performance reasons don't |
| proxmox_vmid | ing | omit | yes* | Having an assigned vmid makes it easier to manage through automation. See host_vars|

Dependencies
------------
````yaml
collections:
  - community.proxmox
python:
  - proxmoxer==2.2.0

````
SSL Certs
------------
When building your OCP cluster, you can add your CA cert as well as your wildcard cert to the initial build. There are a couple of considerations when doing so.  You must put your CA cert in the files/install-config.template file in PEM format.  
You must also include your wildcard cert and key in the files/custom-ingress-certs.yaml file.  Your wildcard cert MUST be in base64 format as a single string within the custom-ingress-certs.yaml file.

To Do
------------
- Convert files to be true jinja templates
- Convert ssl portions of templates to do lookups and convert to base64 during deployment of template
- Convert ipxe files to templates to simplify deployment
- Convert install-config template to a jinja template and use a variable for the pull secret portion of the template

Example Playbook
----------------

````shell
# To deploy a cluster 
ansible-playbook boostrap_ocp_cluster.yml

# To destroy a cluster 
ansible-playbook boostrap_ocp_cluster.yml -t never
````

Known Bugs
-------
Using hotplug for memory has issues. You may need to increase the assigned memory by 1G when calculating. E.g. 1024 * 80 = 81920, but you have to bump it by a 1G to 82944 for hotplug to work with VM provisioning. This is a proxmox issue.


License
-------

MIT

Author Information
------------------

Randy Romero  
binbashroot@gmail.com
