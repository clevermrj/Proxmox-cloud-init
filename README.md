# Proxmox-cloud-init
Hostname automatically updated. 


Prerequisites 
-Download and install an clound image to the host

log into proxmox host.
  
  1. Create New VM
    # qm create 8001 --memory 4000 --cores 2 --name ubuntu-cloud --net0 virtio,bridge=vmbr0,tag=20
  
  2. Import the downloaded Ubuntu disk to local-lvm storage
    # qm importdisk 8001 lunar-server-cloudimg-amd64.img local-lvm
  
  3. Attach the new disk to the vm as a scsi drive on the scsi controller
    # qm set 8001 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-8001-disk-0
  
  4. Add cloud init drive
    # qm set 8001 --ide2 local-lvm:cloudinit
  
  5. Make the cloud init drive bootable and restrict BIOS to boot from disk only
    # qm set 8001 --boot c --bootdisk scsi0
  
  6. Add serial console
    # qm set 8001 --serial0 socket --vga serial0
  

Configure any cloud init settings on the proxmox GUI (user, password, ssh keys, IP config) Regenerate image

Start the VM.
The reason why I am starting the VM because for some reason cloud init uses the template's hostname during DHCP configuration.
Theres little documentation out of there for this issue, so to resolve it I created the script below to restart the systemd-networkd when the VM starts, and then the correct hostname is displayed in DHCP.

  1. Create a Startup_script.sh ( I create this scrip in /usr/local/bin/)
    # echo '#!/bin/bash' | tee /usr/local/bin/startup_script.sh
    # echo 'systemctl restart systemd-networkd' >> /usr/local/bin/startup_script.sh
    # chmod +x /usr/local/bin/startup_script.sh
  2. Create the system service file
    # sudo tee /etc/systemd/system/startup_script.service > /dev/null <<EOF
[Unit]
Description=Execute startup script
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/startup_script.sh

[Install]
WantedBy=multi-user.target
EOF
  3. Reload systemd and start service
    # sudo systemctl daemon-reload
    # sudo systemctl enable startup_script.service

After creating the script we now have to clean the vm and prepare it for cloud init template

  1. clear hostname
    # truncate -s0 /etc/hostname
    # hostnamectl set-hostname localhost

  2. remove files
    # rm /etc/netplan/50-cloud-init.yaml
  3. clearing machine-id
    # truncate -s0 /etc/machine-id
    # rm /var/lib/dbus/machine-id
    # ln -s /etc/machine-id /var/lib/dbus/machine-id

  4. cloud-init clean
    # cloud-init clean
  5. clear shell history
    # truncate -s0 ~/.bash_history
    # history -c
  6. shut down VM
    # shutdown -h now


Now the VM is ready to convert to template



