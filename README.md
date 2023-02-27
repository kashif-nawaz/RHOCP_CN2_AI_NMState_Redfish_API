# RHOCP (Redhat Openshift) Juniper CN2 Installation via AI (Assisted Installer) with NMState and Redfish_API
## Problem Statement
* Juniper Networks CN2 (Cloud Native, telco grade SDN Controller) has been recently certified with (RHOCP) Redhat Open Shift  (Industry leading Container based orchestrator) for telco regional data centers, 5G RAN edge and far edge application and IT Cloud Applications (hosted in public / private clouds).
* Juniper documentation covers deployment of CN2 with RHOCP via AI (Assisted Installer), however that deployment method  does not cover headless deployment i.e. placing boot images at central jumphost from where all the OCP (Openshift) nodes get boot images.
* Rather, that document assumes that boot images are placed/ attached to target OCP nodes via some out of band method.
* Juniper documentation also does not cover how to provide a customized networking config to target OCP nodes (e.g. if port bundle or Link Aggregation or VLAN tagging is required inside OCP target nodes).
## Proposed Solution
* In this wiki I will cover how to manage OCP target nodes via Redfish API (power cycle and adding / removing boot media from central Jumphost via http).
* I will also cover how to provide static network configuration to target OCP nodes via NMState
* Once ISO images were remotely attached to OCP nodes as CD-Rom then their boot order had to be changed to set CD-Rom as 1st boot order.
* But this arrangement introduced another problem with VM based OCP nodes as during OCP installation OCP nodes requires 2-3 reboots and VMs were rebooting  very quickly by loading ISO Image from CD-Rom which is attached from Jumphost over HTTP via Redfish API.
* Although; I was removing remotely attached ISO image from the VM based OCP nodes, but that requires a cold reboot of the VM and during installation I can not do a manual cold reboot VM based OCP nodes during the installation process.
* To solve above problem I had to define  VM based OCP nodes in such a way  that on 1st reboot during the OCP installation OCP nodes (VMs) shall go into a shutdown state rather then restart and before that I had to remove that remotely connected ISO image plus change the boot order from CD to HDD.
## References 

## Topology 
![Topology](./images/topology.png)
## Execution

### Create VMs on KVM Host 
* I am assuming that on the KVM host all required libraries (libvirtd, kvm, virsh etc.) are installed before hand to host the VMs (OCP Nodes).
* Due to paucity of resources I am using single KVM server to host all OCP nodes (3 Controllers and 2 worker nodes).
#### Creating libvirt Storage Pool
```
sudo virsh pool-list --all
sudo mkdir -p /var/lib/libvirt/sushy-host
sudo virsh pool-define-as default --type dir --target /var/lib/libvirt/sushy-host
sudo virsh pool-autostart default
sudo virsh pool-start default
```
#### VM Defination 
```
sudo virt-install -n master01.ocp.pxe.com \
 --description "master01 Machine for Openshift 4 Cluster" \
 --ram=16384 \
 --vcpus=8 \
 --os-type=Linux \
 --os-variant=rhel8.0 \
 --noreboot \
 --disk pool=images,bus=virtio,size=200 \
 --graphics vnc,listen=0.0.0.0  \
 --boot hd,menu=on \
 --events on_reboot=destroy \
 --network bridge=br-ctrplane,mac=52:54:00:8b:a1:17 \
 --network bridge=br-Tenant,mac=52:54:00:8b:a1:18


sudo virt-install -n master02.ocp.pxe.com  \
 --description "Master02 Machine for Openshift 4 Cluster" \
 --ram=16384 \
 --vcpus=8 \
 --os-type=Linux \
 --os-variant=rhel8.0 \
 --noreboot \
 --disk pool=images,bus=virtio,size=200 \
 --graphics vnc,listen=0.0.0.0  \
 --boot hd,menu=on \
 --events on_reboot=destroy \
 --network bridge=br-ctrplane,mac=52:54:00:ea:8b:9d \
 --network bridge=br-Tenant,mac=52:54:00:ea:8b:9c

sudo virt-install -n master03.ocp.pxe.com  \
--description "Master03 Machine for Openshift 4 Cluster" \
--ram=16384 \
--vcpus=8 \
--os-type=Linux \
--os-variant=rhel8.0 \
 --noreboot \
--disk pool=images,bus=virtio,size=200 \
--graphics vnc,listen=0.0.0.0 \
--boot hd,menu=on \
--events on_reboot=destroy \
--network bridge=br-ctrplane,mac=52:54:00:f8:87:c7 \
--network bridge=br-Tenant,mac=52:54:00:f8:87:c8

sudo virt-install -n worker01.ocp.pxe.com \
 --description "worker01 Machine for Openshift 4 Cluster" \
 --ram=16384 \
 --vcpus=8 \
 --os-type=Linux \
 --os-variant=rhel8.0 \
 --noreboot \
 --disk pool=images,bus=virtio,size=200 \
 --graphics vnc,listen=0.0.0.0  \
 --boot hd,menu=on \
 --events on_reboot=destroy \
 --network bridge=br-ctrplane,mac=52:54:00:31:4a:38 \
 --network bridge=br-Tenant,mac=52:54:00:31:4a:39

 sudo virt-install -n worker02.ocp.pxe.com \
 --description "worker02 Machine for Openshift 4 Cluster" \
 --ram=16384 \
 --vcpus=8 \
 --os-type=Linux \
 --os-variant=rhel8.0 \
 --noreboot \
 --disk pool=images,bus=virtio,size=200 \
 --graphics vnc,listen=0.0.0.0 \
 --boot hd,menu=on \
 --events on_reboot=destroy \
 --network bridge=br-ctrplane,mac=52:54:00:6a:37:32 \
 --network bridge=br-Tenant,mac=52:54:00:6a:37:33 
```
#### Configure Sushi-Emulartor on KVM Host 
* Sushi-Emulator will be created as Podman container and later on Podman container can be configured as systemd service
* Hence, I am using Ubuntu 20.04 on my KVM host, so installing Podman was bit challenging on Ubunut but if the KVM host is based on RHEL, Centos or Fedora then installing Podman is fairly straight forward.
* (Link-1)[https://askubuntu.com/questions/1296657/unable-to-install-podman-in-ubuntu-20-04-running-on-wsl2-in-windows-10]
* (Link-2)[https://stackoverflow.com/questions/73942531/podman-unable-to-build-image-from-dockerfile-error-creating-overlay-mount]

```
mkdir -p /etc/sushy/
cat << EOF > /etc/sushy/sushy-emulator.conf
SUSHY_EMULATOR_LISTEN_IP = u'0.0.0.0'
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_SSL_CERT = None
SUSHY_EMULATOR_SSL_KEY = None
SUSHY_EMULATOR_OS_CLOUD = None
SUSHY_EMULATOR_LIBVIRT_URI = u'qemu:///system'
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = False
SUSHY_EMULATOR_BOOT_LOADER_MAP = {
    u'UEFI': {
        u'x86_64': u'/usr/share/OVMF/OVMF_CODE.secboot.fd'
    },
    u'Legacy': {
        u'x86_64': None
    }
}
EOF 
```
#### Creating Sushy-Emulator Container
```
sudo podman create --net host --privileged --name sushy-emulator -v "/etc/sushy":/etc/sushy -v "/var/run/libvirt":/var/run/libvirt "${SUSHY_TOOLS_IMAGE}" sushy-emulator -i :: -p 8000 --config /etc/sushy/sushy-emulator.conf
podman start sushy-emulator
```

### Creating Jumphost VM
* I am using Centos8 as my jumphost ,one can choose any Linux distro. 
```
wget 'https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2'
sudo qemu-img create -b /var/lib/libvirt/images/CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2  -f qcow2 -F qcow2 /var/lib/libvirt/images/ocp_pxe_jumphost.qcow2 200G

sudo virt-customize  -a /var/lib/libvirt/images/ocp_pxe_jumphost.qcow2   --run-command 'xfs_growfs /' --selinux-relabel

cat > ocp_pxe_jumphost_cloud_init.cfg <<END_OF_SCRIPT
#cloud-config
hostname: bastion
fqdn: bastion.ocp.pxe.com
manage_etc_hosts: false
users:
  - name: lab
    groups: wheel
    lock_passwd: false
    shell: /bin/bash
    home: /home/lab
    ssh_pwauth: true
    ssh-authorized-keys:
      - 'ssh-key'
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
chpasswd:
  list: |
     lab:contrail123
  expire: False
write_files:
  - path:  /etc/sudoers.d/lab
    permissions: '0440'
    content: |
         lab ALL=(root) NOPASSWD:ALL
  - path:  /etc/sysconfig/network-scripts/ifcfg-eth0
    permissions: '0644'
    content: |
         TYPE=Ethernet
         PROXY_METHOD=none
         BROWSER_ONLY=no
         BOOTPROTO=none
         IPV4_FAILURE_FATAL=no
         NAME=eth0
         DEVICE=eth0
         ONBOOT=yes
         IPADDR=192.168.24.13
         NETMASK=255.255.255.0
         GATEWAY=192.168.24.1
         DNS1=1.1.1.1
         PEERDNS=yes
runcmd:
 - [sudo, ifup, eth0]
 - [sudo, sed ,-i, 's/PasswordAuthentication no/PasswordAuthentication yes/g', /etc/ssh/sshd_config]
 - [sudo, systemctl, restart, sshd]
END_OF_SCRIPT

cloud-localds -v  ocp_pxe_jumphost_cloud_init.img ocp_pxe_jumphost_cloud_init.cfg


virt-install --name ocp_pxe_jumphost \
  --virt-type kvm --memory 16384  --vcpus 8 \
  --boot hd,menu=on \
  --disk path=ocp_pxe_jumphost_cloud_init.img,device=cdrom \
  --disk path=/var/lib/libvirt/images/ocp_pxe_jumphost.qcow2,device=disk \
  --os-type=Linux \
  --os-variant=rhel8.0 \
  --network bridge:br-ctrplane \
  --graphics vnc,listen=0.0.0.0 --noautoconsole
```
#### Installing Required Packages on Jumphost
* Login to Jumphost.  
```
cd ~/
cat <<EOF> jmphost_setup.sh
sudo dnf --disablerepo '*' --enablerepo=extras swap centos-linux-repos centos-stream-repos -y
sudo dnf distro-sync -y
sudo dnf -y install epel-release
sudo dnf -y install  git dhcp-server  syslinux  httpd   bind bind-utils  vim wget curl bash-completion  tree tar libselinux-python3 firewalld chrony
sudo reboot
EOF
chmod +x jmphost_setup.sh
./jmphost_setup.sh
```
#### Enabling Required Servcies and Firewall Rules on Jumphost 
* Clone this Git Repo
```
git clone https://github.com/kashif-nawaz/RHOCP_CN2_AI_NMState_Redfish_API.git
```
* Preparing named server 
* Copy named.conf ,reverse.db and reverse.db into your jumphost and do required changes as per your setup. 
```
sudo cp -rv ~/RHOCP_CN2_AI_NMState_Redfish_API/named.conf  /etc
sudo cp -rv  ~/RHOCP_CN2_AI_NMState_Redfish_API/reverse.db /var/named/
sudo cp -rv ~/RHOCP_CN2_AI_NMState_Redfish_API/zonefile.db /var/named/
sudo nmcli connection modify eth0 ipv4.dns 127.0.0.1
sudo nmcli connection modify eth0 ipv4.dns-search ocp.pxe.com
sudo nmcli connection modify eth0 ipv4.ignore-auto-dns yes
sudo systemctl restart NetworkManager
```
* Preparing ntp server 
```

sudo vim /etc/chrony.conf
allow 192.168.24.0/24
```
* Preparing Web Server
```
sudo mkdir -p /var/www/html/rhcos
sudo sed -i 's/Listen 80/Listen 0.0.0.0:8080/' /etc/httpd/conf/httpd.conf
```
* Adding Firewall Rules 

```
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --add-service={ntp,http,https,dns} --permanent
sudo firewall-cmd --reload
```
* Enabling , Starting the Services

```
sudo systemctl enable httpd --now
sudo systemctl enable named --now 
sudo systemctl enable chronyd --now 
```

### Preparing OCP AI Installation on Jumphost
* Login to  [Redhat Portal](https://console.redhat.com/openshift/token?ref=cloud-cult-devops)
* ![Click Load Token](./images/offline_token.png)
* Save the token in your system
```
OFFLINE_ACCESS_TOKEN="..."
```
* Export TOKEN into your environment to get access to AI resources.
* This token will expire every 5 minutes, so you need to refresh the following token if any of the steps in subsequent sections get failed due to token expiry.
```
export TOKEN=$(curl \
--silent \
--data-urlencode "grant_type=refresh_token" \
--data-urlencode "client_id=cloud-services" \
--data-urlencode "refresh_token=${OFFLINE_ACCESS_TOKEN}" \
https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token | \
jq -r .access_token)
```
* Verify if AI API is accessible. 
```
curl -s https://api.openshift.com/api/assisted-install/v2/component-versions -H "Authorization: Bearer ${TOKEN}" | jq
```
* Get pull secret from [Redhat Portal](https://console.redhat.com/openshift/install/pull-secret?ref=cloud-cult-devops)
![pull-secret](./images/pull-secret.png)
* Save the pull secret into your system.
* Get access token from Juniper account team to access Juniper Images Registry enterprise.hub.juniper.net.
* Merge the Red Hat Pull-Secret and CN2 Pull Secret into a single file.
* Genrete ssh-key in Jumphost.
* Prepare deployment file. 
```
export ASSISTED_SERVICE_API="https://api.openshift.com"
export CLUSTER_VERSION="4.8"
export CLUSTER_IMAGE="quay.io/openshift-release-dev/ocp-release:4.8.0-x86_64"
export CLUSTER_NAME="ocp"
export CLUSTER_DOMAIN="pxe.com"
export CLUSTER_NET_CIDR="10.128.0.0/14"
export CLUSTER_HOST_PFX="23" 
export CLUSTER_SVC_CIDR="172.31.0.0/16"
export NTP_SOURCE="192.168.24.13"
export CLUSTER_MACHINE_CIDR="192.168.24.0/24"
export CLUSTER_API_VIP=192.168.24.198
export CLUSTER_INGRESS_VIP=192.168.24.199

cat << EOF > ./deployment.json 
{ 
  "kind": "Cluster", 
  "name": "$CLUSTER_NAME", 
  "openshift_version": "$CLUSTER_VERSION", 
  "ocp_release_image": "$CLUSTER_IMAGE", 
  "base_dns_domain": "$CLUSTER_DOMAIN", 
  "hyperthreading": "all", 
  "machine_networks": [{"cidr": "192.168.24.0/24"}],
  "cluster_network_cidr": "$CLUSTER_NET_CIDR", 
  "cluster_network_host_prefix": $CLUSTER_HOST_PFX, 
  "service_network_cidr": "$CLUSTER_SVC_CIDR", 
  "user_managed_networking": false, 
  "vip_dhcp_allocation": false, 
  "additional_ntp_source": "$NTP_SOURCE", 
  "ssh_public_key": "$CLUSTER_SSH_KEY", 
  "pull_secret": $PULL_SECRET 
} 
EOF
```
* Create  OCP Cluster 
```
export CLUSTER_ID=$(curl -s -X POST "$ASSISTED_SERVICE_API/api/assisted-install/v2/clusters" -d @./deployment.json --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" | jq -r '.id')
```
* Verify if the cluster is created. 
```
curl -s -X GET "https://api.openshift.com/api/assisted-install/v2/clusters" -H "accept: application/json" -H "Authorization: Bearer $TOKEN" | jq -r
curl -s -X GET --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -H "get_unregistered_clusters: false" $ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID?with_hosts=true | jq -r '.validations_info' | jq .

curl -s -X GET --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -H "get_unregistered_clusters: false" $ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID?with_hosts=true | jq -r '.status'

curl -s -X GET --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -H "get_unregistered_clusters: false" $ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID?with_hosts=true | jq -r '.validations_info' | jq .
```
* Set CN2 as CNI in  the above created Openshift Cluster

```
curl --header "Content-Type: application/json" --request PATCH --data '"{\"networking\":{\"networkType\":\"Contrail\"}}"' -H "Authorization: Bearer $TOKEN" $ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID/install-config
```
* Verify if Changes are applied

```
curl -s -X GET --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" $ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID/install-config | jq -r
```
* Create CN2 specific Igniation File i.e "infra-ignition.json"

```
{"ignition_config_override": "{\"ignition\":{\"version\":\"3.1.0\"},\"systemd\":{\"units\":[{\"name\":\"ca-patch.service\",\"enabled\":true,\"contents\":\"[Service]\\nType=oneshot\\nExecStart=/usr/local/bin/ca-patch.sh\\n\\n[Install]\\nWantedBy=multi-user.target\"}]},\"storage\":{\"files\":[{\"path\":\"/usr/local/bin/ca-patch.sh\",\"mode\":720,\"contents\":{\"source\":\"data:text/plain;charset=utf-8;base64,IyEvYmluL2Jhc2gKc3VjY2Vzcz0wCnVudGlsIFsgJHN1Y2Nlc3MgLWd0IDEgXTsgZG8KICB0bXA9JChta3RlbXApCiAgY2F0IDw8RU9GPiR7dG1wfSB8fCB0cnVlCmRhdGE6CiAgcmVxdWVzdGhlYWRlci1jbGllbnQtY2EtZmlsZTogfAokKHdoaWxlIElGUz0gcmVhZCAtYSBsaW5lOyBkbyBlY2hvICIgICAgJGxpbmUiOyBkb25lIDwgPChjYXQgL2V0Yy9rdWJlcm5ldGVzL2Jvb3RzdHJhcC1zZWNyZXRzL2FnZ3JlZ2F0b3ItY2EuY3J0KSkKRU9GCiAgS1VCRUNPTkZJRz0vZXRjL2t1YmVybmV0ZXMvYm9vdHN0cmFwLXNlY3JldHMva3ViZWNvbmZpZyBrdWJlY3RsIC1uIGt1YmUtc3lzdGVtIHBhdGNoIGNvbmZpZ21hcCBleHRlbnNpb24tYXBpc2VydmVyLWF1dGhlbnRpY2F0aW9uIC0tcGF0Y2gtZmlsZSAke3RtcH0KICBpZiBbWyAkPyAtZXEgMCBdXTsgdGhlbgoJcm0gJHt0bXB9CglzdWNjZXNzPTIKICBmaQogIHJtICR7dG1wfQogIHNsZWVwIDYwCmRvbmUK\"}}]},\"kernelArguments\":{\"shouldExist\":[\"ipv6.disable=1\"]}}"}
```
* Create infra-envs. json file which will be used to generate ISO images.
```
cat << EOF > ./infra-envs.json
{
  "name": "$CLUSTER_NAME",
  "additional_ntp_sources": "$NTP_SOURCE",
  "ssh_authorized_key": "$CLUSTER_SSH_KEY",
  "pull_secret": $PULL_SECRET,
  "image_type": "full-iso",
  "cluster_id": "$CLUSTER_ID",
  "openshift_version": "$CLUSTER_VERSION",
  "cpu_architecture": "x86_64"
}
EOF
```
* Create Infra-invoice which will be later used patch ignition file and Static Network Config files for OCP nodes. 
```
export INFRA_ENVS_ID=$(curl -s -X POST "$ASSISTED_SERVICE_API/api/assisted-install/v2/infra-envs" -d @infra-envs.json --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" | jq -r '.id')
```
* Patch the infra-ignition. json to infra-envs.

```
curl -s -X PATCH "$ASSISTED_SERVICE_API/api/assisted-install/v2/infra-envs/$INFRA_ENVS_ID" -d @infra-ignition.json --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" | jq -r '.id'
```
* Prepare NMStat config files of OCP nodes [Reference](https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2022/html/assisted_installer_for_openshift_container_platform/assembly_network-configuration#nmstate_configuration)
* I have uploaded master01.yaml  master02.yaml  master03.yaml worker01.yaml  worker02.yaml with wiki, those can be used as reference.
* Assign static MAC to IP addresses, make sure these MACs and NIC names are matched VM/ Physical nodes as per your setup.
```
jq -n --arg NMSTATE_YAML1 "$(cat master01.yaml)" --arg NMSTATE_YAML2 "$(cat master02.yaml)" --arg NMSTATE_YAML3 "$(cat master03.yaml)" --arg NMSTATE_YAML4 "$(cat worker01.yaml)" --arg NMSTATE_YAML5 "$(cat worker02.yaml)" \
'{
  "static_network_config": [
    {
      "network_yaml": $NMSTATE_YAML1,
      "mac_interface_map": [{"mac_address": "52:54:00:8b:a1:17", "logical_nic_name": "ens3"}, {"mac_address": "52:54:00:8b:a1:18", "logical_nic_name": "ens4"}]
    },
    {
      "network_yaml": $NMSTATE_YAML2,
      "mac_interface_map": [{"mac_address": "52:54:00:ea:8b:9d", "logical_nic_name": "ens3"}, {"mac_address": "52:54:00:ea:8b:9c", "logical_nic_name": "ens4"}]
     },
      {
      "network_yaml": $NMSTATE_YAML3,
      "mac_interface_map": [{"mac_address": "52:54:00:f8:87:c7", "logical_nic_name": "ens3"}, {"mac_address": "52:54:00:f8:87:c8", "logical_nic_name": "ens4"}]
     },
      {
      "network_yaml": $NMSTATE_YAML4,
      "mac_interface_map": [{"mac_address": "52:54:00:31:4a:38", "logical_nic_name": "ens3"}, {"mac_address": "52:54:00:31:4a:39", "logical_nic_name": "ens4"}]
     },                                                                                                        
      {
      "network_yaml": $NMSTATE_YAML5,
      "mac_interface_map": [{"mac_address": "52:54:00:6a:37:32", "logical_nic_name": "ens3"}, {"mac_address": "52:54:00:6a:37:33", "logical_nic_name": "ens4"}]
     }
  ]
}' > request-body.json 
```
* Pathc the NMStat config file request-body.json  with the infra-envs.
```
curl -s -X PATCH "$ASSISTED_SERVICE_API/api/assisted-install/v2/infra-envs/$INFRA_ENVS_ID" -d @request-body.json --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" | jq -r '.id'
```
* Generate ISO Image and Download it web server in Jumphost. 
```
export IMAGE_URL=$(curl -H "Authorization: Bearer $TOKEN" -L "$ASSISTED_SERVICE_API/api/assisted-install/v2/infra-envs/$INFRA_ENVS_ID/downloads/image-url" | jq -r '.url')
sudo curl -H "Authorization: Bearer $TOKEN" -L "$IMAGE_URL" -o /var/www/html/rhcos/discovery_image_ocpd.iso
```
* Once Image is downloaed then label SELinux contect to ISO image as per http context.

```
sudo restorecon -RFv /var/www/html/rhcos/
Relabeled /var/www/html/rhcos from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0
Relabeled /var/www/html/rhcos/discovery_image_ocpd.iso from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0
```
* If in case infra-env is required to be deleted.

```
curl -s -X DELETE "$ASSISTED_SERVICE_API/api/assisted-install/v2/infra-envs/$INFRA_ENVS_ID"   --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" | jq '.id'
curl -H "Authorization: Bearer $TOKEN" -L "$ASSISTED_SERVICE_API/api/assisted-install/v2/infra-envs/" | jq .
```


### Verfying VM Access via Redfish API

```
REDFISH_HOST="192.168.24.10"
REDFISH_PORT="8000"
curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/ | jq -r
{
  "@odata.type": "#ComputerSystemCollection.ComputerSystemCollection",
  "Name": "Computer System Collection",
  "Members@odata.count": 6,
  "Members": [
    {
      "@odata.id": "/redfish/v1/Systems/f615f0ac-ea49-4c55-a6e7-1d386cb495cd"
    },
    {
      "@odata.id": "/redfish/v1/Systems/bea822c5-f0c4-43b7-974e-ec1b0945c470"
    },
    {
      "@odata.id": "/redfish/v1/Systems/b667a771-c4b9-457e-899b-8930e7ba1ad1"
    },
    {
      "@odata.id": "/redfish/v1/Systems/9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f"
    },
    {
      "@odata.id": "/redfish/v1/Systems/c57818a6-0551-4007-a6a3-16abc50b2087"
    },
    {
      "@odata.id": "/redfish/v1/Systems/05963fbc-391a-4275-a65f-219e2075150b"
    }
  ],
  "@odata.context": "/redfish/v1/$metadata#ComputerSystemCollection.ComputerSystemCollection",
  "@odata.id": "/redfish/v1/Systems",
  "@Redfish.Copyright": "Copyright 2014-2016 Distributed Management Task Force, Inc. (DMTF). For the full DMTF copyright policy, see http://www.dmtf.org/about/policies/copyright."
}
```
* To get details of any remote host, select its ID and replace it in following command. 
```
curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/05963fbc-391a-4275-a65f-219e2075150b | jq -r
```
* In my case, my interested remote hosts are. 
```
f615f0ac-ea49-4c55-a6e7-1d386cb495cd
bea822c5-f0c4-43b7-974e-ec1b0945c470
b667a771-c4b9-457e-899b-8930e7ba1ad1
9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f
05963fbc-391a-4275-a65f-219e2075150b
```
* From here on I will for loop on these ID to manage my remote hosts via Redfish APIs. 
* Check if CD-ROm is inserted into target remote hosts , it should return "false" value. 
```
for REDFISH_MANAGER in f615f0ac-ea49-4c55-a6e7-1d386cb495cd bea822c5-f0c4-43b7-974e-ec1b0945c470 b667a771-c4b9-457e-899b-8930e7ba1ad1 9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f 05963fbc-391a-4275-a65f-219e2075150b
do 
curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Managers/$REDFISH_MANAGER/VirtualMedia/Cd/ | jq  '[{iso_connected: .Inserted}]'
done 
```
* To insert CD-Rom  and verify it,  run following itteritation, at this stage the ISO image must have been downloaded in placed in web server on Jumphost.

```
for REDFISH_MANAGER in f615f0ac-ea49-4c55-a6e7-1d386cb495cd bea822c5-f0c4-43b7-974e-ec1b0945c470 b667a771-c4b9-457e-899b-8930e7ba1ad1 9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f 05963fbc-391a-4275-a65f-219e2075150b
do 
curl -d \
    '{"Image":"'"$ISO_URL"'", "Inserted": true}' \
     -H "Content-Type: application/json" \
     -X POST \
     http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Managers/$REDFISH_MANAGER/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia
done 

for REDFISH_MANAGER in f615f0ac-ea49-4c55-a6e7-1d386cb495cd bea822c5-f0c4-43b7-974e-ec1b0945c470 b667a771-c4b9-457e-899b-8930e7ba1ad1 9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f 05963fbc-391a-4275-a65f-219e2075150b
do 
curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Managers/$REDFISH_MANAGER/VirtualMedia/Cd/ | jq  '[{iso_connected: .Inserted}]'
done 
```
* To change the boot order from the HDD to CD executes the following iteration.

```
for REDFISH_SYSTEM in f615f0ac-ea49-4c55-a6e7-1d386cb495cd bea822c5-f0c4-43b7-974e-ec1b0945c470 b667a771-c4b9-457e-899b-8930e7ba1ad1 9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f 05963fbc-391a-4275-a65f-219e2075150b
do 

curl -s "http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/$REDFISH_SYSTEM" | jq .Boot
done 


for REDFISH_SYSTEM in f615f0ac-ea49-4c55-a6e7-1d386cb495cd bea822c5-f0c4-43b7-974e-ec1b0945c470 b667a771-c4b9-457e-899b-8930e7ba1ad1 9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f 05963fbc-391a-4275-a65f-219e2075150b
do 

curl -X PATCH -H 'Content-Type: application/json' -d '{"Boot": {"BootSourceOverrideTarget": "Cd","BootSourceOverrideEnabled": "Continuous"}}' "http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/$REDFISH_SYSTEM" | jq .
done 

for REDFISH_SYSTEM in f615f0ac-ea49-4c55-a6e7-1d386cb495cd bea822c5-f0c4-43b7-974e-ec1b0945c470 b667a771-c4b9-457e-899b-8930e7ba1ad1 9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f 05963fbc-391a-4275-a65f-219e2075150b
do 

curl -s "http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/$REDFISH_SYSTEM" | jq .Boot
done 
``` 
* To Start the all the VMs in the target KVM host. 

```
for REDFISH_SYSTEM in f615f0ac-ea49-4c55-a6e7-1d386cb495cd bea822c5-f0c4-43b7-974e-ec1b0945c470 b667a771-c4b9-457e-899b-8930e7ba1ad1 9fa8ae2b-5a4c-4048-a7c6-a95d342fc00f 05963fbc-391a-4275-a65f-219e2075150b
do 

curl -s -d '{"ResetType":"ForceOff"}'  \
    -H "Content-Type: application/json" -X POST  \
    http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/$REDFISH_SYSTEM/Actions/ComputerSystem.Reset

curl -s -d '{"ResetType":"ForceOn"}' \
    -H "Content-Type: application/json" -X POST \
    http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/$REDFISH_SYSTEM/Actions/ComputerSystem.Reset
done 

```
* After some time machines will be registered with OCP cluster, and role (master or worker) can be set for each node accordingly.
#### OCP Installation
* Set API VIPs. 
```
curl -X PATCH "$ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID" -H "accept: application/json" -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -d "{ \"api_vip\": \"$CLUSTER_API_VIP\", \"ingress_vip\": \"$CLUSTER_INGRESS_VIP\"}"
```
* Prepare CN2 Manifests.
* Download [CN2 manifests](https://support.juniper.net/support/downloads/?p=contrail-networking) files and prepare these files for further uploaeding  with the OCP Cluster.
![CN2](./images/cn2_manifests.png)
```
 curl  'https://cdn.juniper.net/software/contrail/22.4.0/contrail-manifests-openshift-22.4.0.284.tgz?.....' -o contrail-manifests-openshift-22.4.0.284.tgz
 tar xvf contrail-manifests-openshift-22.4.0.284.tgz 
 tar xvf contrail-manifests-openshift-22.4.0.284.tgz
 cd ~/contrail-manifests-openshift
 mkdir -p manifests
 cp  -rv *.yaml manifests/
 cp -rv cert-manager-1.8/*.yaml manifests/
 cp -rv vrrp/*.yaml manifests/
 cd ~/contrail-manifests-openshift/manifests
```
* Change CN2 data/ control plan network as per your enviornment 
```
sed -i -e 's/10.40.1/192.168.5/g' 99-network-configmap.yaml
```
* The following step is required as we are using 2 network deployment, which is also called user-managed-network and separate files will be used to disable checksum on VM based OCP Nodes.
```
 grep -rni ens3
 99-disable-offload-master.yaml:20:          ExecStart=/sbin/ethtool -K ens3 tx off
 99-disable-offload-worker.yaml:20:          ExecStart=/sbin/ethtool -K ens3 tx off
 rm -f 99-disable-offload-master.yaml
 rm -f 99-disable-offload-worker.yaml
```
* Prepare script to upload manifests with the cluster. 
```
cd ~/contrail-manifests-openshift
cat > prepare-cn2-manifests.sh <<END_OF_SCRIPT

#!/bin/bash  
echo
echo "Creating manifests for CLUSTER_ID: ${CLUSTER_ID}"
echo
MANIFESTS=(manifests/*.yaml)
total=${#MANIFESTS[@]}
i=0
for file in "${MANIFESTS[@]}"; do
    i=$(( i + 1 ))
    eval "CONTRAIL_B64_MANIFEST=$(cat $file | base64 -w0)";
    eval "BASEFILE=$(basename $file)"; 
    eval "echo '{\"file_name\":\"$BASEFILE\", \"folder\":\"manifests\", \"content\":\"$CONTRAIL_B64_MANIFEST\"}' > $file.b64";
    printf "\nProcessing file: $file\n"
    curl -s -X POST "$ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID/manifests" -d @$file.b64 --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN"
done
echo
echo
echo "Total Manifests: $total"
echo
echo "Manifest List:"
curl -s -X GET "$ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID/manifests" --header "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" | jq -r
END_OF_SCRIPT 
chmod +x prepare-cn2-manifests.sh
./prepare-cn2-manifests.sh 
```
* Installation 
```
curl -X POST "$ASSISTED_SERVICE_API/api/assisted-install/v2/clusters/$CLUSTER_ID/actions/install" -H "accept: application/json" -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" | jq
```

#### Power Cycle Control for VM Based OCP Nodes

#### 