The easiest way to convert images or virtual machines from one format to another is with [virt-v2v](https://libguestfs.org/virt-v2v.1.html). This knowledgebase article will cover how to use virt-v2v to convert VMware images for use in OpenStack.  Virt-v2v can read Linux and Windows guests running on VMware, Xen, Hyper-V and some other hypervisors, and convert them to KVM managed by libvirt, OpenStack, oVirt, Red Hat Virtualisation (RHV) or several other targets. It can modify the guest to make it bootable on KVM and install virtio drivers so it will run quickly.

[Installing virt-v2v](#install)  
[Converting images to OpenStack an compatible image](#converting-images)  
[Converting an image and shrinking the file system](#convert-ovavmx-to-qemukvm-and-shrink-file-system)  
[Using VMware's OVFTool](#vmwares-ovf-tool)

  
Install
----------

You can install virt-v2v with a package manager such as yum:

```bash
sudo yum install virt-v2v virtio-win
```

Note that if you are going to convert any Windows VMs/images, you will also need to install the virtio drivers so that networking will work within OpenStack.  If you have the virtio-win package installed, virt-v2v will actually install the virtio drivers in the image for you.  On other distros (such as Ubuntu) virt-v2v might be part of the libguestfs-tools package:

```bash
sudo apt install libguestfs-tools
```

You may need to install the virtio-win package as well.  It can be found [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.noarch.rpm) for CentOS/RHEL/Fedora. On the air-gapped development networks, you may need to install the virtio drivers from the-pit (Software/v/virtio/virtio-win-1.9.16-2.el7.noarch.rpm) if they aren't available from the repos.  Download the rpm and install with:

```bash
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.noarch.rpm
sudo rpm -i virtio-win.noarch.rpm
```

  
Converting Images
--------------------

Converting is a simple call of virt-v2v and telling it what input mode (-i) which tells it what foreign hypervisor to convert from, what output mode (-o) here we are using "local" to write to a local disk image, the storage location (-os) for the converted image -when using "-o local" this is the directory you want the new image created in, and the output format (-of) -for OpenStack you'll want to use raw or qcow2 output format.

```bash
virt-v2v -i vmx /data/vm_name.vmx -o local -os /data -of raw
```

\*For VMware images, you'll need the image to be either an OVA or the raw VM files (which will include a vmdk and vmx - and you'll point to and specify the vmx file format).  By default, newer versions of vCenter will export VMs as an OVF (this will include the vmdk, ovf and possibly a mf file).  If you simply put these files into a zip or tar file, virt-v2v can convert with the ova input format. If you are the one exporting the VMs from vCenter or requesting them and want them to be a .ova file, you can use [PowerCLI](https://developer.vmware.com/powercli), connect to your vCenter (or ESXi Host) using "Connect-VIServer <vCenter/ESXi name>" and run this command to export as an OVA:

```bash
Get-VM “VM name” | Export-VApp -Destination “<folder name\file name>” -Format OVA
```

**Again, You can just zip or tar up the folder containing the ovf, vmdk, and mf and point virt-v2v to that using "-i ova"**

![](https://i.imgur.com/xlUT6m2.png)

Note that virt-v2v did not find any virtio drivers to install for my Windows VM.  I don't have a virtio-win directory in /usr/share.  To remedy this I do as suggested above -downloading the latest virtio-win rpm and installing it.  You can find the latest version of virtio-win for CentOS [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.noarch.rpm).  You can download and install it with the following, which will create and populate the /usr/share/virtio-win directory with the virtio drivers:

```bash
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.noarch.rpm
sudo rpm -i virtio-win.noarch.rpm
```

I re-ran my virt-v2v conversion and you can see that it installed the virtio drivers for me this time:

![](https://i.imgur.com/xgsA653.png)

You'll see that virt-v2v created the -sda and .xml files - the names of which are based on the VM name.  You can see the name of the VM in the .vmx or .ovf file.  The -sda file (or .qcow2 if you're converting to qcow2 image format) is the image file you can now import into OpenStack through the GUI, or you can use the OpenStack CLI using:

```bash
openstack image create --file /data/vm_name-sda --disk-format raw --project Projectname Imagename
```

That's it!  Now you can build a new instance in OpenStack from that image.  Though, we can do better - we can make that image a lot smaller...

Convert OVA/VMX to QEMU/KVM and shrink file system
--------------------------------------------------

Commonly, the VMs you are converting & importing into OpenStack have a lot of free space on them, and this can result in say a 20GB usage Windows VM that has a 60GB virtual HDD creating a 60GB image that you are importing into OpenStack.  This wastes storage space in OpenStack, and also might require you to use a larger flavor instance in OpenStack than you really need, wasting project resources.  To resolve this, we can shrink the filesystem of the image we're converting before importing it into OpenStack.  
First, we'll convert the image to a compressed qcow2 image **note that you can't use raw for the output here, it needs to be qcow2 to use --compressed.**  

```bash
virt-v2v --compressed -i ova /data/vm.ova -o local -os /data -of qcow2
```

![](https://i.imgur.com/VcyWYB4.png)

**Note that compared to the prior conversion, my qcow2 image here is only 6.8G instead of 40G.**

Now, we will retrieve file system info from the disk image we created. --long is going to display extra columns of data (long format) --parts will display partitions, --blkdevs will display block devices, -h or --human-readable display sizes in human-readable format, -a or --add specifies the disk image.  If the VM has multiple block devices, you must supply them all with separate -a options.

```bash
virt-filesystems --long --parts --blkdevs -h -a vm-sda
```

![](https://i.imgur.com/fmw8DD9.png)

Next, we will check the disk usage on the partition using virt-df:

```bash
virt-df -h vm-sda
```

![](https://i.imgur.com/Bw509om.png)

In this example, we can see that we're only using 14G of that 40G.  We'll give a little buffer and make our image 15G.

We're going to use the [guestfish](https://libguestfs.org/guestfish.1.html) tool to edit the filesystem.  Guestfish is a guest filesystem shell for editing virtual machine filesystems and disk images.  Now the exact commands here will differ based on the filesystem type.  Here is an example for a linux VM:

```bash
guestfish -a vm-sda
	<fs> run
        <fs> list-filesystems
	<fs> e2fsck-f /dev/sda1
	<fs> resize2fs-size /dev/sda1 15G
	<fs> e2fsck-f /dev/sda1
	<fs> quit
```

And here's an example on a Windows VM (NTFS):

```bash
guestfish -a vm-sda
	<fs> run
	<fs> list-filesystems
	<fs> ntfsresize /dev/sda1 size:15G
```

![](https://i.imgur.com/1PispPd.png)

I re-ran the virt-df command to see the new size:

![](https://i.imgur.com/ZaEgX1l.png)

Next, we'll create a new qcow2 image which we will resize our original one to:

```bash
qemu-img create -f qcow2 -o preallocation=metadata newdisk.qcow2 15G
```

Then, we will use virt-resize to shrink our disk:

```bash
virt-resize --shrink /dev/sda1 ./vm-sda ./newdisk.qcow2
```

![](https://i.imgur.com/yRMgXYR.png)

I encountered the above error.  I created the newdisk.qcow2 image at 15GB as well, and it looks like QEMU and ntfsresize calculate that 15GB a little differently.  I'll run the "qemu-img create" command from a step ago again with 16GB this time and run the virt-resize again:  
![](https://i.imgur.com/ZAncI6B.png)

Another error.  Because we resized the partition, NTFS wants to run a chkdsk to make sure everything is ok.  We're going to let that happen upon first boot and we're not booting up any VMs as a part of this process.  So, if you get this error, you can pass --ntfsresize-force to virt-resize to have it ignore that error.

![](https://i.imgur.com/M9zdk7A.png)

If you were doing this for a linux VM/image, sorry for the detour.  
Ok, now let's take a look at that new disk image:

```bash
qemu-img info ./newdisk.qcow2
```

Let's do the virt-filesystems and virt-df checks on it as well:

```bash
virt-filesystems --long --parts --blkdevs -h -a ./newdisk.qcow2
virt-df -h ./newdisk.qcow2
```

![](https://i.imgur.com/jiHix2j.png)

Everything looks good!  We're finished and can upload the new disk image to OpenStack.  We cut off a little over half the size in this example, but I've seen VMs with hundreds of GB sized virtual disks that are still only using ~20GB of space and could save a ton of space using this method.  So, there we have the entire process completed without having to boot into a VM with VirtualBox, virt-manager, VMware Workstation, or anything like that to resize and then re-export the VM.  Everything being command-line means we could also put this entire process into a script...I haven't done that, you didn't think it'd be that easy, did you?

  
VMware's OVF Tool
--------------------

  
You may need to convert a VM from an OVF file format to an OVA (virt-v2v supports OVA or VMX format).  As mentioned previously, you should just be able to zip or tar up the files, but if you need them converted to an OVA for whatever reason, you can use VMware's OVF Tool.  You can download the OVF Tool for your OS here: [https://customerconnect.vmware.com/downloads/get-download?downloadGroup=OVFTOOL443](https://customerconnect.vmware.com/downloads/get-download?downloadGroup=OVFTOOL443).  It does require a VMware account, but not any licenses/products attached to your account, so if you don't have one, just create a new account.  To install the tool on linux, simply make the bundle executable and then run it.  Accept the EULA and it will install the tool.

```bash
sudo chmod +X ./VMware-ovftool-4.4.3-18663434-lin.x86_64.bundle
sudo ./VMware-ovftool-4.4.3-18663434-lin.x86_64.bundle
```

To convert to an OVA, simply give ovftool the source file and destination file desired:

```bash
ovftool /path_to_ovf/vm.ovf /path_to_new_ova_file/vm.ova
```

![](https://i.imgur.com/6y4KJKE.png)
