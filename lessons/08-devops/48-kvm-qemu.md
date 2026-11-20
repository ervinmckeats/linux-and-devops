# Lesson 48. Virtualization: KVM and QEMU

## Goal
Run and manage virtual machines on your own Linux host, and understand what a cloud VM is underneath.

## Theory

- **Hypervisor**: software that runs VMs. **Type 1** runs on bare metal (KVM, ESXi, Xen, Hyper-V); **type 2** runs inside an OS (VirtualBox, VMware Workstation).
- **KVM** is a Linux kernel module that turns Linux into a type 1 hypervisor using CPU virtualization extensions (Intel VT-x / AMD-V).
- **QEMU** emulates the rest of the machine (disks, network cards, BIOS) and uses KVM for fast CPU execution.
- **libvirt** is the management layer: `virsh`, `virt-install`, `virt-manager`, Terraform and Ansible talk to it.
- **virtio** drivers are paravirtual devices (disk, network) that are much faster than emulated hardware.

Most public clouds run on KVM-based hypervisors. Proxmox VE and OpenStack are built on KVM too.

## Check and install

```bash
grep -cE 'vmx|svm' /proc/cpuinfo
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst cloud-image-utils
sudo usermod -aG libvirt,kvm $USER
virsh list --all
```

- `grep -cE 'vmx|svm' /proc/cpuinfo`: > 0 = hardware virtualization available
- `sudo usermod -aG libvirt,kvm $USER`: then re-login

Nested virtualization (a VM inside your lab VM) needs to be enabled on the outer hypervisor; on a laptop it is often easier to run KVM directly on a Linux host.

## Create a VM from a cloud image (fast way)

Cloud images boot in seconds and are configured by **cloud-init**, exactly like a cloud VM.

```bash
cd /var/lib/libvirt/images
sudo wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
sudo qemu-img create -f qcow2 -F qcow2 -b noble-server-cloudimg-amd64.img vm1.qcow2 20G

cat > user-data <<'YAML'
#cloud-config
hostname: vm1
users:
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-key
package_update: true
packages: [qemu-guest-agent]
runcmd:
  - systemctl enable --now qemu-guest-agent
YAML
sudo cloud-localds seed-vm1.iso user-data

sudo virt-install --name vm1 --memory 2048 --vcpus 2 \
  --disk vm1.qcow2,bus=virtio --disk seed-vm1.iso,device=cdrom \
  --os-variant ubuntu24.04 --network network=default,model=virtio \
  --import --noautoconsole

virsh domifaddr vm1
ssh admin@<ip>
```

- `virsh domifaddr vm1`: find its IP

`qemu-img create -b` makes a **copy-on-write overlay**: the base image stays untouched, each VM stores only its own changes.

## Daily management with virsh

```bash
virsh list --all
virsh start vm1; virsh shutdown vm1; virsh destroy vm1
virsh reboot vm1
virsh autostart vm1
virsh console vm1
virsh dominfo vm1
virsh edit vm1
virsh setmem vm1 4G --config; virsh setvcpus vm1 4 --config --maximum
virsh snapshot-create-as vm1 before-upgrade
virsh snapshot-list vm1
virsh snapshot-revert vm1 before-upgrade
virsh undefine vm1 --remove-all-storage
```

- `virsh start vm1; virsh shutdown vm1; virsh destroy vm1`: destroy = power off, not delete
- `virsh console vm1`: serial console (Ctrl+] to exit)
- `virsh edit vm1`: XML definition
- `virsh undefine vm1 --remove-all-storage`: delete for good

## Disks

```bash
qemu-img info vm1.qcow2
qemu-img resize vm1.qcow2 +10G
qemu-img convert -O raw vm1.qcow2 vm1.raw
virsh attach-disk vm1 /var/lib/libvirt/images/data.qcow2 vdb --persistent --subdriver qcow2
```

- `qemu-img resize vm1.qcow2 +10G`: then growpart + resize2fs inside (lesson 19)

qcow2 supports snapshots, thin provisioning and backing files; raw is slightly faster.

## Networking

- `default` network: NAT, VMs get `192.168.122.x` from libvirt's dnsmasq; reachable from the host only.
- **Bridge** to a physical interface: VMs appear on your LAN with their own IPs.

```bash
virsh net-list --all
virsh net-dhcp-leases default
```

## Performance notes

- Use virtio for disk and network, `cpu mode='host-passthrough'` for best CPU performance.
- Do not overcommit memory heavily; ballooning helps but swap on the host kills all VMs.
- **Steal time** (`st` in `top` inside the VM) = the hypervisor is giving your vCPU to someone else. In the cloud that means a noisy neighbour or a burstable instance out of credits.

## Containers vs VMs

| | VM | Container |
|---|---|---|
| Kernel | own | shared with host |
| Boot | seconds to minutes | milliseconds |
| Isolation | strong (hardware-level) | weaker (namespaces, cgroups) |
| Density | tens per host | hundreds per host |
| Use | different OS/kernel, strong tenant isolation, legacy apps | microservices, CI jobs, dense packing |

Lightweight VMs (Firecracker, Kata Containers) combine both: VM isolation with container speed. AWS Lambda and Fargate run on Firecracker.

## Practice

1. Install KVM/libvirt on a Linux host (or a VM with nested virtualization).
2. Create two VMs from the Ubuntu cloud image with cloud-init and SSH keys.
3. Snapshot one, break it (`rm -rf /etc/ssh`), revert.
4. Resize a VM disk by 10 GB and grow the filesystem inside.
5. Use this setup as your lab for the Ansible lesson: manage both VMs with one playbook.

## Check yourself

- What does KVM do and what does QEMU do?
- What is a qcow2 backing file and why does it save space?
- What does high steal time inside a VM tell you?

## Next
[Lesson 49. Kubernetes architecture and core objects](../09-kubernetes/49-k8s-architecture.md)
