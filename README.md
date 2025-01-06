# MicroVM with Firecracker

The following tries to create MicroVM (AWS lambda function) with Firecracker locally on x86_64 Linux device. Below is the experiment environment:

```
Hardware: Normal 8 core Intel, 16GB RAM, 500GB HDD, Ubuntu 20.04 LTS
```

```
Software: KVM with QEMU emulator version 4.2.1, Firecracker 1.7.0
 (there exists newer version but that causes a problem mentioned later).
 MicroVM is compiled with Linux kernel 5.10.225, filesystem is ubuntu 22.04. 
```

## Reading sources

Original source: [Firecracker getting started](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md). It's recommeded to read to understand how it works and to modify the system following your specific needs.

Straightfoward to setup a running code inside MicroVM: [Qxf2 Blog](https://qxf2.com/blog/build-a-microvm-using-firecracker/).

## Instruction

This instruction provides a straight and quick guideline how you can quickly download Firecracker, create a custom microVM with your code inside and run it successfully. Further explaination can be read in the reading sources, especially the second one, which I followed also.

### First step: Download Firecracker

Just go to the Firecracker github and grab the [v.1.7.0](https://github.com/firecracker-microvm/firecracker/releases) version, the name is: **firecracker-v1.7.0-x86_64.tgz**.

Extract it with cmd: `tar -xvzf firecracker-v1.7.0-x86_64.tgz`

There will be multiple files in the extracted folder, you just need to care the file named **firecracker-v1.7.0-x86_64**. Rename it and put in in bin folder so that it can be called at any directory in your device.

```bash
mv firecracker-v1.7.0-x86_64 /usr/local/bin/firecracker

```

Now the Firecracker is ready to manage your created MicroVM.

### Second step: Create a custom MicroVM with your code inside

Clone the Firecracker repo

```bash
git clone https://github.com/firecracker-microvm/firecracker.git
```

You just need some config files and scripts from this repo, but just clone the entire repo for the sake of simplicity. Now, you have three files/folders equal to three sub-steps that need to be done.

1. `/resources/rebuild.sh`: this file builds the roofts file and the kernel. The roofts file can be understood as your Ubuntu hard drive, with the Ubuntu OS installed and your custom code inside.
2. `/resources/chroot.sh`: this file defines what your Ubuntu system should have, for example you need to have python3 and its relevant packages to run your custom code, you need to setup network interfaces for your VM, etc.
3. `/resources/overlay`: this folder will be copied into your VM, thus you can put your scripts, systemd service here so that your settings and app can automatically run up when the VM is booted up.

Start first with 1, the chroot.sh file. Here just put your required packages/code compilers for your VM to run your code or testing purpose in the line

```
packages="udev systemd-sysv openssh-server iproute2 curl socat python3-minimal python3-venv python3-pip git iperf3 iputils-ping fio kmod tmux hwloc-nox vim-tiny trace-cmd linuxptp strace"
```

For example, here I need python3-venv for my code, iperf3 to test the throughput of the VM. Then you can proceed to insert cmd to download your code and install dependencies, for example I download my simple web app and install the python packages to run it.

```bash
mkdir -p /home/projects
git clone https://github.com/kienkauko/simple_web.git /home/projects/simple_web
source /home/venv/simple_web/bin/activate
python -m pip install -r /home/projects/simple_web/requirements.txt
deactivate
```

Just think about this step as RUN cmds in Dockerfile.

In the next step 2, you can copy any scripts, services you want to start along with the VM. For example, I put my web_app.py as a service by adding the file [simple_web.service](firecracker/resources/overlay/etc/systemd/system/simple_web.service) to firecracker/resources/overlay/etc/systemd/system. It is better to follow the instruction from [Qxf2 Blog](https://qxf2.com/blog/build-a-microvm-using-firecracker/). In this blog, they also put the networking setup script for the VM in [fcnet.service](firecracker/resources/overlay/etc/systemd/system/fcnet.service) and [fcnet-setup.sh](firecracker/resources/overlay/usr/local/bin/fcnet-setup.sh) to route traffic to the network TAP to connect to the host Internet interface. This is mentioned in the big Third step further below.

Then, just proceed to run the step 3, the rebuild.sh file. Currently, the current file from Firecracker repo does not create a ext4 filesystem for the VM. The custom [rebuild_custom.sh](firecracker/resources/rebuild_custom.sh) file from [Qxf2 Blog](https://qxf2.com/blog/build-a-microvm-using-firecracker/) has already included it. To build Ubuntu 22.04 and Linux kernel 5.10.225, we run it with the following cmd:

```bash
sudo ./rebuild_custom.sh -f -k 5.10
```

Before running the above script, it is recommended to enable `CONFIG_PCI=y` in the [resources/guest_configs/microvm-kernel-ci-x86_64-5.10.config](resources/guest_configs/microvm-kernel-ci-x86_64-5.10.config) . This will prevent the error when boosting up VM later. It's reported as a [bug](https://github.com/firecracker-microvm/firecracker/issues/4881) that only happens when compiling kernel 6.1 from non-AWS github repo. But many people also found this error with different kernel version and Firecracker version. That is why I stick with kernel 5.10 and Firecracker 1.7.0 :)

#### Common errors here:

* No space left on device. If this error happens, that mean you assign too little space for the filesystem while your libraries require more. Just increase the roofts size inside the script to the amount you think is enough: local SIZE=${3:-400M} current amount is 400MB.
* Manifest is not found. This error may come from aprevious failure of a command. It could come from apt install "your_package" failed to install/find a specified package.

The building process runs successfully when it outputs a tree like this:

```
[4.0K]   /home/xxxx/projects/firecracker/resources/x86_64
├── [1.8M]  initramfs.cpio
├── [400M]  ubuntu-22.04.ext4
├── [2.5K]  ubuntu-22.04.id_rsa
├── [5.7K]  ubuntu-22.04.manifest
├── [ 60M]  ubuntu-22.04.squashfs
├── [ 16M]  vmlinux-5.10.225
├── [ 56K]  vmlinux-5.10.225.config
└── [ 62K]  vmli
```

### Third step: Network setup, check JSON config file and boost up the VM.

Setup a network TAP to connects the VM with the physical network in case you want your VM to have Internet. There is other way such as setting up NAT, this can be found Firecracker networking documentations.

Again, provides a nice script to get off all the burden. You just need to run this script before boosting up your VM[ tap_setup.sh](firecracker/tap_setup.sh)

Be aware that I need to change the network interface in the file to your Internet-facing interface of the physical device. For example, I put **enp0s25** in all cmds as it is my PC main interface.

```bash
sudo iptables -t nat -D POSTROUTING -o enp0s25 -j MASQUERADE || true
```

Now, 99% of the job has been done, you just need to change some contexts in the [vm_config.json](firecracker/vm_config.json) file. For example, the path to the kernel: "kernel_image_path": "vmlinux-5.10.225", path to the rootfs file: "path_on_host": "ubuntu-22.04.ext4", Internet-facing id "iface_id": "enp0s25", allocated CPU and MEM: "vcpu_count": 1,    "mem_size_mib": 512.

Everything is done, just boost up your MicroVM and test if everything works as intended.

`sudo firecracker --config-file vm_config.json`

If you encounter the error /dev/root: Can't open blockdev while booting then probably it is the [ACPI bug](https://github.com/firecracker-microvm/firecracker/issues/4881) I have mentiond above. Try to compile lower version Kernel with lower version Firecracker.
