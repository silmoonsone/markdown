Set-VMNetworkAdapter -VMName "Ubuntu Server" -MacAddressSpoofing On

apt install -y qemu-kvm libvirt-daemon-system virtinst bridge-utils cockpit cockpit-machines


systemctl enable --now libvirtd
systemctl enable --now cockpit
systemctl start cockpit.socket
systemctl status cockpit.socket

usermod -aG libvirt $USER
usermod -aG kvm $USER



virsh list --all