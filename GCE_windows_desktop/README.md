# gce_windows_desktop
This is a tutorial on how to create a windows desktop on a GCE instance using GCE nested vm feature

## Prep
1. [Create a google cloud platform account and project](https://cloud.google.com/free/)
2. [Install GcloudSDK](https://www.terraform.io/intro/getting-started/install.html)
3. [Enable GCP APIs](https://support.google.com/cloud/answer/6158841?hl=en) (Enable Compute Engine API, Google Clous Storage)
4. Clone or download the repo

##Set Environment variable needed
'''
GCP_Project=<Project name>
DISK_NAME=<name vm disk>
ZONE=<desired zone>
'''

##Configure GcloudSDK
Please use the CloudSDK on your local machine or the cloud shell in the UI on Google Cloud Platform
'''
Gcloud init
'''

##Create a base virtual disk from ubuntu-16.04
'''
gcloud compute --project=$GCP_Project disks create $DISK_NAME --zone=$ZONE --image=ubuntu-1604-lts-drawfork-v20180810 --image-project=eip-images --size=20GB
'''

##Create a custom nested vm base image

'''
gcloud compute images create nested-vm-image \
  --source-disk $DISK_NAME --source-disk-zone $Zone\
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"
'''      

##Create a VM instance
Create a VM instance with the custom image in a zone that supports Haswell or higher chips.  ALter the <VM_Name_Here> with the name of the vm you would like to use.
'''
gcloud compute instances create <VM_Name_Here> --zone $ZONE \
              --image nested-vm-image
'''

##Confirm that nested virtualization is enabled in the VM.
1. Connect to the VM instance. For example:
'''
gcloud compute ssh example-nested-vm
'''
Or within the GCP consol click on the ssh icon next to you GCE instance.

2. Check that nested virtualization is enabled by running the following command. A non-zero response confirms that nested virtualization is enabled.
'''
grep -cw vmx /proc/cpuinfo
'''

3. Update the VM instance and install some necessary packages:
'''
sudo apt-get update && sudo apt-get install qemu-kvm -y && sudo apt-get install libvirt-bin -y

sudo apt-get update && sudo apt-get install uml-utilities qemu-kvm bridge-utils virtinst libvirt-bin -y
'''
## Copy and convert you windows images to a .qcow2 format
 Create a local image folder and GCS bucket and Upload your windows desktop image to a GCS bucket using the command line (replace <Bucketname> with the name of your desired bucket name:
'''
mkdir images
cd images
gsutil mb gs://<Bucketname>/
'''
This needs to be run from the local machine or you can manaually load the image using the google cloud storage UI
'''
gsutil cp <windows_image_location> gs://<Bucketname>
'''

Switch back to the ssh terminal of the nested VM and copy the image to the images folder on the nested VM.  You should be in the images folder you previously created.  Please make sure to keep the same file extention on the file when you copy it.  I suggest using the same file name if possible.
'''
gsutil cp gs://<Bucketname>/<Objectname> <Objectname>
'''

We are now going to use QEMU to convert our windows image to a .qcow2 format which is a file system from KVM.  In this case we will covert a .vmdk but this process supports a number of image conversions.  
###qemu-img strings
'''
QCOW2(KVM, Xen)=qcow2
QED(KVM)=qed
raw=raw
VDI(VirtualBox)=vdi
VHD(Hyper-V)=vpc
VMDK(VMware)=vmdk
'''
1. Convert the image from vmdk to qcow2:
'''
qemu-img convert -f vmdk -O qcow2 <Objectname>.vmdk <Objectname>.qcow2
'''
confirm the name .qcow2 image is there with an "ls" command

2. Run screen:
'''
screen
'''
Hit enter at the screen welcome prompt.

3. 
'''
sudo qemu-system-x86_64 -enable-kvm -hda win7.qcow2 -m 512 -curses
'''
virsh qemu-monitor-command --hmp windows7 'hostfwd_add ::13389-:3389