# GCE_windows_desktop
This is a tutorial on how to create a windows desktop on a GCE instance using GCE nested vm feature

## Prep
1. [Create a google cloud platform account and project](https://cloud.google.com/free/)
2. [Install GcloudSDK](https://www.terraform.io/intro/getting-started/install.html)
3. [Enable GCP APIs](https://support.google.com/cloud/answer/6158841?hl=en) (Enable Compute Engine API, Google Clous Storage)
4. Clone or download the repo

## Set Environment variable needed
```
GCP_PROJECT=<Project name>
DISK_NAME=<name vm disk>
ZONE=<desired zone>
```

## Configure GcloudSDK
Please use the CloudSDK on your local machine or the cloud shell in the UI on Google Cloud Platform
```
Gcloud init
```

## Create a base virtual disk from ubuntu-16.04
```
gcloud compute --project=$GCP_PROJECT disks create $DISK_NAME --zone=$ZONE --image=ubuntu-1604-lts-drawfork-v20180810 --image-project=eip-images --size=30GB
```

## Create a custom nested vm base image

```
gcloud compute images create nested-vm-image \
  --source-disk $DISK_NAME --source-disk-zone $ZONE\
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"
```     

## Create a VM instance
Create a VM instance with the custom image in a zone that supports Haswell or higher chips[confirm here](https://cloud.google.com/compute/docs/regions-zones/).  Alter the <VM_Name_Here> with the name of the vm you would like to use.
```
gcloud compute instances create <VM_Name_Here> --zone $ZONE \
              --image nested-vm-image
```

## Confirm that nested virtualization is enabled in the VM.
1. Connect to the VM instance. For example:
```
gcloud compute ssh example-nested-vm
```
Or within the GCP console click on the ssh icon next to your GCE instance.

2. Check that nested virtualization is enabled by running the following command. A non-zero response confirms that nested virtualization is enabled.
```
grep -cw vmx /proc/cpuinfo
```

3. Update the VM instance and install some necessary packages:
```
sudo apt-get update && sudo apt-get install qemu-kvm -y && sudo apt-get install bridge-utils

```

4. Edit network interface
We are going to create a bridge interface that will allow the nested vm to communicate with the desired networks and use the vm host level for all firewall and security admnistration.
```
cd /etc/network/
sudo nano interfaces
```
Add the following to the bottom of the interfaces file

```
auto br0
iface br0 inet dhcp
   bridge_ports eth0 eth1
   bridge_stp off
   bridge_fd 0
   bridge_maxwait 0
```

Save the file and then restrart the newtork services by running this command ( Please ignore any errors and this may take 3-5 minutes to complete)
```
sudo /etc/init.d/networking restart 
```

To confirm the network interface (br0) has been created run:
```
ip addr
```
You should see the br0 interface listed here.

## Create qemu bridge configuration

We need to create a bridge configuration file so that Qemu knows what bridge networks it can connect to.
```
sudo mkdir /etc/qemu
cd /etc/qemu
sudo nano bridge.conf
```
once in the file paste this single line
```
allow br0
```
save the file

## Copy and convert you windows images to a .qcow2 format
 Create a local image folder and GCS bucket and Upload your windows desktop image to a GCS bucket using the command line, replace <Bucketname> with the name of your desired bucket: 
```
cd
mkdir images
cd images
gsutil mb gs://<Bucketname>/
```
This needs to be run from the local machine or you can manaually load the image using the google cloud storage UI
```
gsutil cp <windows_image_location> gs://<Bucketname>
```

Switch back to the ssh terminal of the nested VM and copy the image to the images folder on the nested VM.  You should be in the images folder you previously created.  Please make sure to keep the same file extention on the file when you copy it.  I suggest using the same file name if possible.
```
gsutil cp gs://<Bucketname>/<Objectname> <destination_on_vm>
```

We are now going to use QEMU to convert our windows image to a .qcow2 format which is a file system from KVM.  In this case we will covert a .vmdk but this process supports a number of image conversions. 

### Qemu-img strings
These are the file types qemu is able to convert
```
QCOW2(KVM, Xen) = qcow2
QED(KVM) = qed
raw = raw
VDI(VirtualBox) = vdi
VHD(Hyper-V) = vpc
VMDK(VMware) = vmdk
```

## Build and deploy your image
1. Convert the image from vmdk to qcow2:
```
qemu-img convert -f vmdk -O qcow2 <Objectname>.vmdk <Objectname>.qcow2
```
confirm the name .qcow2 image is there with an "ls" command

2. Run the converted windows image 
```
sudo qemu-system-x86_64 -enable-kvm -cpu host -m 1024 -net nic -net bridge,br=br0 -vnc :0 <filepath/image_file_name>
```

## GCP VPC network 
You will need to create a vpc firewall rule to allow the vnc connection from outside of google cloud.
In this case i have deployed to the "default" vpc and the rule can be created by the following command.
```
gcloud compute --project=$GCP_PROJECT firewall-rules create sample-qemu-vnc --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:5900 --source-ranges=0.0.0.0/0
```

## It worked!
Use the public IP address of the GCE instance and the port :5900 (the default vnc port for qemu) to vnc to the windows desktop nested vm you just created.  You can use any vnc viewer for this part. (I used realvnc)
example:
```
0.0.0.0:5900
```

# What to know and Future
Feedback on this would be appreciated and i will be optimizing this in a more scripted fasion utilizing Terraform to scale this approach.
