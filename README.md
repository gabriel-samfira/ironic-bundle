# Post Deployment Config

## Create network

```bash
openstack network create \
	--share \
	--provider-network-type flat \
	--provider-physical-network physnet2 \
	deployment

openstack subnet create \
	--network deployment \
	--dhcp \
	--subnet-range 10.13.0.0/16 \
	--gateway 10.13.0.1 \
	--ip-version 4 \
	--allocation-pool start=10.13.2.1,end=10.13.2.255 \
	deployment

export NETWORK_ID=$(openstack network show deployment --format json | jq -r .id)
```

## Create a keypair

```bash
openstack keypair create --public-key $HOME/.ssh/id_rsa.pub mykey
```

## Building the images

### Install prerequisites

```bash
pip3 install --user diskimage-builder ironic-python-agent-builder
```

### Build the deploy images

These images will be used to PXE boot the bare metal node for installation

Create a folder for the images:

```bash
export IMAGE_DIR="$HOME/images"
mkdir -p $IMAGE_DIR
```


```bash
ironic-python-agent-builder ubuntu \
	-o $IMAGE_DIR/ironic-python-agent
```

### Build the baremetal images

These images will be deployed to the bare metal server.

Generate Bionic and Focal images:

```bash
for release in bionic focal
do
    export DIB_RELEASE=$release
    disk-image-create --image-size 5 \
        ubuntu vm dhcp-all-interfaces \
	iscsi-boot block-device-efi \
        -o $IMAGE_DIR/baremetal-ubuntu-$release
done
```

Let's break down the above command:
  * ubuntu - builds an Ubuntu image
  * [vm](https://docs.openstack.org/diskimage-builder/latest/elements/vm/README.html) - the image will be a "whole disk" image
  * [dhcp-all-interfaces](https://docs.openstack.org/diskimage-builder/latest/elements/dhcp-all-interfaces/README.html) - will use DHCP on all interfaces
  * [block-device-efi](https://docs.openstack.org/diskimage-builder/latest/elements/block-device-efi/README.html) - creates a GPT partition table, suitable for booting an EFI system
  * [iscsi-boot](https://docs.openstack.org/diskimage-builder/latest/elements/iscsi-boot/README.html) - Handles configuration for the disk to be capable of serving as a remote root filesystem through iSCSI


### Upload images to Glance

Convert images to raw. Not necessarily needed, but this will save CPU cycles at deployment time: 

```bash
for release in bionic focal
do
    qemu-img convert -f qcow2 -O raw \
	$IMAGE_DIR/baremetal-ubuntu-$release.qcow2 \
	$IMAGE_DIR/baremetal-ubuntu-$release.img
    rm $IMAGE_DIR/baremetal-ubuntu-$release.qcow2
done
```

Upload OS images. Operating system images need to be uploaded to the swift backend if we plan to use direct deploy mode:

```bash
for release in bionic focal
do
    glance image-create \
        --store swift \
        --name baremetal-$release \
        --disk-format raw \
        --container-format bare \
        --file $IMAGE_DIR/baremetal-ubuntu-bionic.img --progress
done
```

Upload deployment images:

```bash
glance image-create \
    --store swift \
    --name deploy-vmlinuz \
    --disk-format aki \
    --container-format aki \
    --visibility public \
    --file $IMAGE_DIR/ironic-python-agent.kernel --progress

glance image-create \
    --store swift \
    --name deploy-initrd \
    --disk-format ari \
    --container-format ari \
    --visibility public \
    --file $IMAGE_DIR/ironic-python-agent.initramfs --progress
```

Save the image IDs as variables for later:

```bash
export DEPLOY_VMLINUZ_UUID=$(openstack image show deploy-vmlinuz --format json| jq -r .id)
export DEPLOY_INITRD_UUID=$(openstack image show deploy-initrd --format json| jq -r .id)
```

## Create flavors for bare metal

Flavor characteristics like memory and disk are not used for scheduling. Disk size is used to determine de root partition size of the bare metal node. If in doubt, make the DISK_GB variable smaller than the size of the disks you are deploying to. Cloud-init will take care of expanding the partition on first boot.

```bash
# Match these to your HW
export RAM_MB=4096
export CPU=4
export DISK_GB=6
export FLAVOR_NAME="baremetal-small"
```

Create flavor and set resource class. We will add the same resource class to the node we will be enrolling later. The scheduler will use the resource class to find a node that matches the flavor:

```bash
openstack flavor create --ram $RAM_MB --vcpus $CPU --disk $DISK_GB $FLAVOR_NAME
openstack flavor set --property resources:CUSTOM_BAREMETAL_SMALL=1 $FLAVOR_NAME
```

Disable scheduling based on standard flavor properties:

```bash
openstack flavor set --property resources:VCPU=0 $FLAVOR_NAME
openstack flavor set --property resources:MEMORY_MB=0 $FLAVOR_NAME
openstack flavor set --property resources:DISK_GB=0 $FLAVOR_NAME
```

## Enroll a node

Create the node and save the UUID:

```bash
export NODE_NAME01="ironic-node01"
export NODE_NAME02="ironic-node02"
openstack baremetal node create --name $NODE_NAME01 \
	--driver ipmi \
	--deploy-interface direct \
	--raid-interface agent \
	--driver-info ipmi_address=10.10.0.1 \
	--driver-info ipmi_port=6230 \
	--driver-info ipmi_username=admin \
	--driver-info ipmi_password=Passw0rd \
	--driver-info deploy_kernel=$DEPLOY_VMLINUZ_UUID \
	--driver-info deploy_ramdisk=$DEPLOY_INITRD_UUID \
	--driver-info cleaning_network=$NETWORK_ID \
	--driver-info provisioning_network=$NETWORK_ID \
	--property capabilities='boot_mode:bios' \
	--resource-class baremetal-small \
	--property cpus=4 \
	--property memory_mb=4096 \
	--property local_gb=20

openstack baremetal node create --name $NODE_NAME02 \
        --driver ipmi \
        --deploy-interface direct \
        --raid-interface agent \
        --driver-info ipmi_address=10.10.0.1 \
        --driver-info ipmi_port=6231 \
        --driver-info ipmi_username=admin \
        --driver-info ipmi_password=Passw0rd \
        --driver-info deploy_kernel=$DEPLOY_VMLINUZ_UUID \
        --driver-info deploy_ramdisk=$DEPLOY_INITRD_UUID \
        --driver-info cleaning_network=$NETWORK_ID \
        --driver-info provisioning_network=$NETWORK_ID \
        --resource-class baremetal-small \
        --property capabilities='boot_mode:uefi' \
        --property cpus=4 \
        --property memory_mb=4096 \
        --property local_gb=25


export NODE_UUID01=$(openstack baremetal node show $NODE_NAME01 --format json | jq -r .uuid)
export NODE_UUID02=$(openstack baremetal node show $NODE_NAME02 --format json | jq -r .uuid)
```

Create a port for the node. The MAC address must match the MAC address of the nic attached to the baremetal server. Make sure to map the port to the proper physical network:

```bash
openstack baremetal port create 52:54:00:6a:79:e6 \
	--node $NODE_UUID01 \
	--physical-network=physnet2

openstack baremetal port create 52:54:00:c5:00:e8 \
        --node $NODE_UUID02 \
        --physical-network=physnet2

```

## Make node available for deployment

```bash
openstack baremetal --os-baremetal-api-version 1.11 node manage $NODE_UUID01
openstack baremetal --os-baremetal-api-version 1.11 node provide $NODE_UUID01

openstack baremetal --os-baremetal-api-version 1.11 node manage $NODE_UUID02
openstack baremetal --os-baremetal-api-version 1.11 node provide $NODE_UUID02
```

## Boot form volume

Set the node storage interface to cinder:

```bash
openstack baremetal node set $NODE_NAME02 \
	--storage-interface cinder
```

Create a volume to boot from:

```bash
export VOLUME_NAME="$NODE_NAME02-volume"
openstack volume create \
	--image baremetal-bionic \
	--size 25 $VOLUME_NAME
volume=$(openstack volume show $VOLUME_NAME --format json | jq -r .id)
```

Node: Wait for the image to become "available"

Create an iSCSI connector for the node:

```bash
openstack --os-baremetal-api-version 1.33 baremetal volume connector create \
          --node $NODE_UUID02 --type iqn --connector-id iqn.2017-08.org.openstack.$NODE_UUID02
```

And finally, boot the machine:

```bash
openstack server create \
	--flavor baremetal-small \
	--volume $volume \
	--key-name mykey test
```
