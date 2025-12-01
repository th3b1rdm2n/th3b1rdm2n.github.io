---
author: "Bírd Màn"
title: "Automating Virtual Machine Deployment for Research"
date: "2025-10-04"
description: "Learn how to automate the deployment of virtual machines for research purposes using CentOS, Debian, and Ubuntu with the virsh command and custom configuration files."
summary: "Streamline your research environment setup by automating deployment of CentOS, Debian, and Ubuntu virtual machines with minimal manual intervention."
tags: ["Technology", "Automation"]
categories: ["Engineering"]
aliases: ["automating-virtual-machine-deployment-for-research"]
thumbnail: "images/banner-automating-virtual-machine-deployment-for-research.webp"
---

Setting up virtual machines (VMs) interactively can be a tedious task. In this article, we’ll walk through how to automate the creation of VMs for CentOS, Debian, and Ubuntu using a simple script. This method saves time and ensure consistency across your research infrastructure.

We have developed a bash script with `install_virtualization_packages` function to install virtualization packages, another function `setup_storage_pools` to define and create storage folders where our ISO and disk files are stored, and finally `deploy_virtual_machine` function which automates the deployment of virtual machines with minimal manual interaction. The `deploy_virtual_machine` function uses `virt-install` for creating and managing VMs on Linux-based systems using KVM (Kernel-based Virtual Machine) and libvirt, `cloud-init`, `preseed` and `kickstart` files to streamline the process of configuring the VMs with specific settings.

#### The `install_virtualization_packages` Function

Before creating virtual machines, we need to ensure our host system has all the required virtualization tools installed. The `install_virtualization_packages` function detects the distribution and installs the appropriate packages. It sets up libvirt, KVM, and supporting utilities like `virt-manager` and `cloud-image-utils`, then restarts the virtualization service.

```bash
install_virtualization_packages() {
  # Define core variables
  local pkgs service_name distro

  # Detect the distro name from /etc/os-release, or exit if unavailable
  if [ -f /etc/os-release ]; then
    distro=$(grep -oP '(?<=^ID=).*' /etc/os-release | tr -d '"')
  else
    echo "Cannot detect distribution."
    return 1
  fi

  # Install virtualization packages based on detected distro
  case "$distro" in
    ubuntu|debian)
      # Core virtualization/GUI packages on Debian/Ubuntu
      pkgs="virt-manager bridge-utils libosinfo-bin libvirt-daemon-system libvirt-clients qemu-kvm cloud-image-utils"
      service_name="libvirtd"
      sudo apt-get -y update
      sudo apt-get install -y $pkgs
      ;;
    fedora|rhel|centos)
      # Install Fedora/RHEL/CentOS virtualization group
      pkgs="virt-viewer qemu-kvm libvirt libvirt-daemon"
      service_name="libvirtd"
      sudo dnf -y update
      sudo dnf install -y $pkgs
      ;;
    arch)
      # Install virtualization packages on Arch
      pkgs="virt-manager libvirt qemu edk2-ovmf"
      service_name="libvirtd"
      sudo pacman -Sy --noconfirm $pkgs
      ;;
    opensuse*|suse)
      # Install virtualization packages on openSUSE
      pkgs="virt-manager libvirt-daemon libosinfo"
      service_name="libvirtd"
      sudo zypper install -y $pkgs
      ;;
    *)
      echo "Unsupported distribution: $distro"
      return 1
      ;;
  esac

  # Add user to libvirt and kvm groups
  sudo usermod -aG libvirt $USER

  # Restart the virtualization service
  sudo systemctl restart "$service_name"

  echo "Virtualization tools installed. Service restarted. You may need to re-login for group changes."
}
```

_How it Works_  

- **Distribution Detection**: It first detects the host's Linux distribution using /etc/os-release.

- **Package Installation**: Based on the distribution (ubuntu/debian, fedora/rhel/centos, arch, opensuse), it defines the appropriate list of packages (virt-manager, qemu-kvm, libvirt-daemon-system, cloud-image-utils, etc.) and uses the corresponding package manager (apt-get, dnf, pacman, zypper) to install them.

- **User Group Membership**: The function adds the current user ($USER) to the libvirt group. This is crucial for managing virtual machines as a non-root user without needing to prefix every virsh or virt-install command with sudo.

- **Service Management**: It restarts the core virtualization service (libvirtd) to ensure all new configurations and group memberships are active. A note is provided to the user that they may need to re-login for the group changes to take full effect.

#### The `setup_storage_pools` Function

The next step is configuring storage locations for VM ISO and disk files. The `setup_storage_pools` function sets up persistent libvirt storage pools, creating separate directories for images and machines under a base folder (`/media/$USER/CARD`).

```bash
setup_storage_pools() {
  # Define user, group, and base directory
  local user="$(id -un)"
  local group="$(id -gn)"
  local base_dir="/media/${user}/CARD"

  # Fix ownership and permissions on base directory or exit if missing
  if [[ -d "$base_dir" ]]; then
    sudo chown -R "$user:$group" "$base_dir"
    sudo chmod -R u+rwX "$base_dir"
  else
    echo "Base directory $base_dir does not exist. Is the CARD drive mounted?"
    return 1
  fi

  # Define storage pools and paths
  declare -A pools=(
    ["card_images"]="${base_dir}/images"
    ["card_machines"]="${base_dir}/machines"
  )

  # Create and activate libvirt storage pools
  for name in "${!pools[@]}"; do
    local path="${pools[$name]}"
    mkdir -p "$path"

    if ! sudo virsh pool-info "$name" &>/dev/null; then
      sudo virsh pool-define-as "$name" dir --target "$path"
    fi

    if ! sudo virsh pool-info "$name" 2>/dev/null | grep -q "Active:.*yes"; then
      sudo virsh pool-start "$name"
    fi

    sudo virsh pool-autostart "$name"
  done

  sudo virsh pool-list --all
}
```

_How it Works_  

- **Directory Management**: It defines a base_dir (assumed to be an external drive/mount point: /media/$USER/CARD) and verifies its existence. It also fixes the ownership and permissions of this directory to ensure the current user can access it.

- **Pool Definitions**: It uses an associative array to map desired libvirt pool names (like card_images and card_machines) to their corresponding physical directories.

- **Libvirt Pool Creation**: For each defined pool:
It creates the physical directory if it doesn't already exist. Then it uses sudo virsh pool-define-as to define a new storage pool if one with that name doesn't exist. This registers the directory with the libvirt daemon.

- **Pool Activation**: It checks if the pool is active (Active: yes) and, if not, starts it using sudo virsh pool-start. 

- **Autostart Configuration**: It configures the pool to automatically start on host boot with sudo virsh pool-autostart.

- **Verification**: Finally, it lists all pools to allow the user to verify the configuration. This ensures that VM ISOs and disk images are stored in locations that libvirt is aware of and can manage.


#### The `deploy_virtual_machine` Function
```bash
deploy_virtual_machine() {
  # Define core VM variables
  local name iso_image disk_size ram vcpus os_variant network_name
  local username="" password="" root_password=""
  
  # Define paths variables
  local base_dir="/media/$(id -un)/CARD"
  local image_store="${base_dir}/images"
  local machine_store="${base_dir}/machines"
  local seed_iso

  if [[ "$1" =~ ^(-h|--help)$ ]]; then
    cat <<EOF
Usage: $FUNCNAME -n <name> -i <iso_image> -u <username> -p <password> [-d <disk_size>] [-r <ram>] [-c <vcpus>] [-o <os_variant>]

Required Flags:
  -n,  --name           Name of the VM
  -i,  --iso            ISO image filename in ${image_store}
  -u,  --username       Username
  -p,  --password       Password (prompt if not given)
  -rp, --root-password  Root Password (prompt if not given)

Optional Flags:
  -d, --disk-size   Disk size (default: 20G)
  -r, --ram         RAM in MB (default: 2048)
  -c, --cpu         Number of vCPUs (default: 2)
  -o, --os-variant  OS variant string for virt-install
  -net, --network   Network name (default: default)
  -h, --help        Show this help
EOF
    return 0
  fi

  while [[ $# -gt 0 ]]; do
    case $1 in
      -n|--name)       name="$2"; shift 2 ;;
      -i|--iso)        iso_image="$2"; shift 2 ;;
      -d|--disk-size)  disk_size="$2"; shift 2 ;;
      -r|--ram)        ram="$2"; shift 2 ;;
      -c|--cpu)        vcpus="$2"; shift 2 ;;
      -u|--username)   username="$2"; shift 2 ;;
      -p|--password)   password="$2"; shift 2 ;;
      -rp|--root-password)   root_password="$2"; shift 2 ;;
      -o|--os-variant) os_variant="$2"; shift 2 ;;
      -net|--network)  network_name="$2"; shift 2 ;;
      *) echo "Unknown option: $1"; return 1 ;;
    esac
  done

  # Set defaults if argument is no provided
  disk_size=${disk_size:-20G}
  ram=${ram:-2048}
  vcpus=${vcpus:-2}
  network_name=${network_name:-default}
  seed_iso="${machine_store}/${name}-seed.iso"

  # Validate passed inputs
  [[ -z "$name" || -z "$iso_image" ]] && { echo "name and iso_image are required"; $FUNCNAME -h; return 1; }
  
  # Prompt for user password
  while password=$(echo "$password" | xargs) && [[ -z "$password" ]]; do
    read -s -p "Enter password for $username: " password
    echo
    [[ -z "$password" ]] && echo "Password cannot be empty. Please try again."
  done
  
  # Prompt for root password
  while root_password=$(echo "$root_password" | xargs) && [[ -z "$root_password" ]]; do
    read -s -p "Enter root password: " root_password
    echo
    [[ -z "$root_password" ]] && echo "Root password cannot be empty. Please try again."
  done

  # Ensure directories exist and define file paths
  mkdir -p "$image_store" "$machine_store"
    
  # Validate iso file exists
  local iso_path="${image_store}/${iso_image}"
  [[ ! -f "$iso_path" ]] && { echo "ISO not found at $iso_path"; return 1; }

  # Auto-detect OS variant
  detect_os_variant() {
    local iso_file="$1"
    local detected_os
    detected_os=$(osinfo-detect "$iso_file" 2>/dev/null | grep -oP "(?<=Media is an installer for OS ')[^']+")
    detected_os=${detected_os%% (*}
    detected_os=${detected_os/ Server/}
    detected_os=${detected_os/ Desktop/}
    detected_os=$(echo "$detected_os" | xargs)
  
    [[ -n "$detected_os" ]] && 
    osinfo-query os --fields=short-id,name | tail -n +3 | awk -F'|' -v search="$detected_os" '
      tolower($2) ~ tolower(search) { gsub(/^[ \t]+|[ \t]+$/, "", $1); print $1; exit }
    ' || echo "generic"
  }
  os_variant=$(detect_os_variant "$iso_path")
  echo "Detected OS variant: $os_variant"

  # Detect OS family
  local os_family=""
  if [[ "$os_variant" =~ (ubuntu) ]]; then
    os_family="ubuntu"
  elif [[ "$os_variant" =~ (debian) ]]; then
    os_family="debian"
  elif [[ "$os_variant" =~ (rhel|centos|almalinux|fedora) ]]; then
    os_family="rhel"
  else
    echo "Unknown OS variant, defaulting to Ubuntu-like autoinstall"
    os_family="ubuntu"
  fi
  
    # Define and verify disk file or created
    local disk_path="${machine_store}/${name}.qcow2"
    [[ ! -f "$disk_path" ]] && {
    sudo qemu-img create -f qcow2 -o cluster_size=2M "$disk_path" "$disk_size" || return 1
    sudo chown libvirt-qemu:libvirt-qemu "$disk_path"
    sudo chmod 660 "$disk_path"
  }

  # SSH key generation
  local ssh_dir="$HOME/.ssh"
  local ssh_key="${ssh_dir}/${name}"
  local ssh_pub_key="${ssh_key}.pub"
  if [[ ! -f "$ssh_key" || ! -f "$ssh_pub_key" ]]; then
    echo "Generating SSH key for $name"
    ssh-keygen -t ed25519 -f "$ssh_key" -N "" -C "${username}@${name}" || return 1
  fi

  # Create installation config based on OS family
  local tmpdir
  tmpdir=$(mktemp -d)
  trap "rm -rf '$tmpdir'" EXIT INT TERM
  
  # Read in the ssh key and store as a string
  local ssh_public_key=$(<"$ssh_pub_key")
  
  # Hash the user password for distros that use it
  local user_passwd_hash="$(openssl passwd -6 "${password}")"
  # local user_passwd_hash=$(mkpasswd -m sha-512 -s <<< "$password")
  
  # Hash the root password for distros that use it
  local root_passwd_hash="$(openssl passwd -6 -stdin <<< "${root_password}")"
  
  case "$os_family" in
    ubuntu)
      local user_data="${tmpdir}/user-data"
      local meta_data="${tmpdir}/meta-data"
      cat > "$user_data" <<EOF
#cloud-config
autoinstall:
  version: 1
  interactive-sections: []

  # Set hostname, default username, and hashed password
  identity:
    hostname: ${name}
    username: ${username}
    password: "${user_passwd_hash}"

  # Install and enable SSH with the provided public key
  ssh:
    install-server: true
    authorized_keys: []
    allow-pw: yes

  # Use entire disk with automatic LVM layout
  storage:
    layout:
      name: lvm

  # Install a small set of useful packages in addition to the base system
  updates: all
  packages: [vim, curl, wget, openssh-server]

  # Configure language, locale, and system timezone
  locale: en_US.UTF-8
  timezone: Africa/Lagos
  user-data:
    disable_root: false
    users:
      - name: root
        passwd: ${root_passwd_hash}
        lock_passwd: false
    runcmd:
      - bash -c "mkdir -p /home/${username}/.ssh && echo '${ssh_public_key}' > /home/${username}/.ssh/authorized_keys && chmod 600 /home/${username}/.ssh/authorized_keys && chown -R ${username}:${username} /home/${username}/.ssh"
      - apt-get install -y qemu-guest-agent spice-vdagent -qq
      - systemctl enable --now ssh
      - systemctl start qemu-guest-agent spice-vdagent
EOF
      cat > "$meta_data" <<EOF
instance-id: $name
local-hostname: $name
EOF
      # cloud-localds "$seed_iso" "$user_data" "$meta_data" || return 1
      genisoimage -output "$seed_iso" -volid cidata -joliet -rock "$user_data" "$meta_data"  || return 1
      ;;

    debian)
      local debian_codename="" apt_mirror="mirror.litnet.lt" preseed="${tmpdir}/preseed.cfg"
      if [[ "$os_variant" =~ debian([0-9]+) ]]; then
        case "${BASH_REMATCH[1]}" in
          13) debian_codename="trixie" ;;
          12) debian_codename="bookworm" ;;
          11) debian_codename="bullseye" ;;
          10) debian_codename="buster" ;;
          *) debian_codename="stable" ;;
        esac
      fi
      echo "Using Debian Suite: $debian_suite"
      cat > "$preseed" <<EOF
# Language and System Selection
d-i debian-installer/language string en
d-i debian-installer/country string NG
d-i debian-installer/locale string en_US.UTF-8

# Keyboard Selection
d-i console-setup/ask_detect boolean false
d-i keyboard-configuration/xkb-keymap select us

# Date and Time Selection 
d-i clock-setup/utc boolean true
d-i time/zone string Africa/Lagos

# Auto-select network interface, set hostname, and configure timezone
d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string ${name}

# Account Setup
d-i passwd/root-login boolean true
d-i passwd/root-password-crypted password ${root_passwd_hash}
d-i passwd/user-fullname string ${username}
d-i passwd/username string ${username}
d-i passwd/user-password-crypted password ${user_passwd_hash}

# Use LVM with the atomic partitioning recipe
d-i partman-auto/method string lvm
d-i partman-auto/choose_recipe select atomic

# Automatically remove existing LVM volumes
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true

# Automatically remove existing RAID (md) arrays
d-i partman-md/device_remove_md boolean true
d-i partman-md/confirm boolean true

# Finalize partitioning and allow changes to disk labels
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true

# Disable CD-ROM as a source
d-i apt-setup/disable-cdrom-entries boolean true

# Enable APT mirror without specifying a country
d-i apt-setup/use_mirror boolean true
d-i mirror/http/mirror string deb.debian.org
d-i mirror/http/directory string /debian
d-i mirror/protocol string http

# Enable components and services
d-i apt-setup/services-select multiselect security, updates
d-i apt-setup/contrib boolean true
d-i apt-setup/non-free boolean true
d-i apt-setup/non-free-firmware boolean true

# Disable package selection
d-i pkgsel/run_tasksel boolean false

# GRUB installation
d-i grub-installer/only_debian boolean true
d-i grub-installer/bootdev string default

# Configure sudo, install essentials, and set up SSH
d-i preseed/late_command string echo "${username} ALL=(ALL:ALL) ALL" > /target/etc/sudoers.d/users; \
  echo -e "deb https://${apt_mirror}/debian/ ${debian_codename} main non-free-firmware\ndeb https://${apt_mirror}/debian/ ${debian_codename}-updates main\ndeb https://${apt_mirror}/debian-security/ ${debian_codename}-security main non-free-firmware" > /target/etc/apt/sources.list; \
  in-target apt-get update -y; \
  in-target apt-get install -y git openssh-server spice-vdagent -qq; \
  in-target mkdir -p /home/${username}/.ssh; \
  echo '${ssh_public_key}' > /target/home/${username}/.ssh/authorized_keys; \
  in-target chmod 600 /home/${username}/.ssh/authorized_keys; \
  in-target chmod 700 /home/${username}/.ssh; \
  in-target chown -R ${username}:${username} /home/${username}/.ssh; \
  in-target systemctl enable sshd; \
  in-target systemctl start sshd; \
  in-target systemctl start spice-vdagentd

# Suppresses confirmation on installation completion and enable automatic reboot
d-i finish-install/reboot_in_progress note
EOF
      genisoimage -output "$seed_iso" -volid cidata -joliet -rock "$preseed" || return 1
      ;;

    rhel)
      local kickstart="${tmpdir}/ks.cfg"
      cat > "$kickstart" <<EOF
#version=DEVEL
graphical
firstboot --disable

# Language, Keyboard and Timezone
lang en_US.UTF-8
keyboard us
timezone --utc Africa/Lagos

# Network settings
network --bootproto dhcp --onboot yes --activate --hostname=${name}

# Security policies
authselect select local
selinux --permissive

# Enable SSH for remote access 
firewall --enabled --ssh

# Setup user accounts and passwords(system hashes the password itself)
rootpw --allow-ssh --iscrypted ${root_passwd_hash}
user --name=${username} --iscrypted --password=${user_passwd_hash} --groups=wheel

# Wipe all existing partitions, initialize label, and use LVM for automatic partitioning.
clearpart --all --initlabel
autopart --type=lvm

# Install required package groups and utilities
%packages
@^Server with GUI
@development
@network-tools
curl
wget
openssh-server
qemu-guest-agent
spice-vdagent
%end

# Post-install configuration
%post --log=/var/log/kickstart_post.log

# Add SSH public key for the user
mkdir -p /home/${username}/.ssh
echo "$(cat "$ssh_pub_key")" > /home/${username}/.ssh/authorized_keys
chmod 600 /home/${username}/.ssh/authorized_keys
chmod 700 /home/${username}/.ssh
chown -R ${username}:${username} /home/${username}/.ssh

# Grant sudo privileges to the user
echo "${username} ALL=(ALL:ALL) ALL" >> /etc/sudoers.d/${username}
chmod 0440 /etc/sudoers.d/${username}

# Enable and start SSH service
systemctl enable sshd
systemctl start sshd

# Start guest agents (qemu/spice)
systemctl start qemu-guest-agent
systemctl start spice-vdagentd
%end

# Reboot after installation
reboot
EOF
      # NOTE: Kickstart files are typically accessed via HTTP, but we use a CDROM
      genisoimage -output "$seed_iso" -volid cidata -joliet -rock "$kickstart" || return 1
      ;;
  esac
  
  # Initialize network
  echo "Ensuring libvirt network '$network_name' is started and enabled..."
  sudo virsh net-start "$network_name" 2>/dev/null || true
  sudo virsh net-autostart "$network_name"

  # Determine how to boot the installer for each distro family
  local extra_args=""
  local install_source=()
  
  if [[ "$os_family" == "ubuntu" ]]; then
    # 100%
    install_source=(--location "$iso_path,kernel=casper/vmlinuz,initrd=casper/initrd" 
      --initrd-inject="$user_data" --initrd-inject="$meta_data")
    extra_args="quiet autoinstall ds=nocloud\;s=/cdrom/"
  elif [[ "$os_family" == "debian" ]]; then
    install_source=(--location "$iso_path" --initrd-inject="$preseed")
    extra_args="auto=true priority=critical preseed/file=/preseed.cfg"
  elif [[ "$os_family" == "rhel" ]]; then
    # 100%
    install_source=(--location "$iso_path")
    extra_args="inst.ks=cdrom:/ks.cfg"
  fi

  echo "Launching VM '$name' with OS variant '$os_variant'..."
  # Build virt-install command dynamically
  virt_install_cmd=(
    sudo virt-install
    --name "$name"
    --memory "$ram"
    --vcpus "$vcpus"
    --disk path="$disk_path",format=qcow2,bus=virtio
    --disk path="$seed_iso",device=cdrom
    "${install_source[@]}"
    --os-variant "$os_variant"
    --graphics vnc
    --network network=${network_name},model=virtio
    --noautoconsole
    --hvm
    --virt-type kvm
    --autostart
  )
  
  # Append --extra-args only if non-empty
  if [[ -n "$extra_args" ]]; then
    virt_install_cmd+=(--extra-args "$extra_args")
  fi
  
  # Run the command
  "${virt_install_cmd[@]}" || { echo "VM installation failed."; return 1; }

  # start virtual manager
  virt-manager -c qemu:///system & 
  
  # Wait for shutdown, then cleanup seed ISO
  echo "Waiting for VM '$name' to shut down before removing seed ISO..."
  while true; do
    state=$(sudo virsh domstate "$name" 2>/dev/null)
    if [[ "$state" == "shut off" ]]; then
      sudo virsh domblklist $name
      sudo virsh detach-disk $name $seed_iso --config --persistent
      echo "Cleaning up seed ISO: $seed_iso"
      rm -f $seed_iso
      sudo chown "$USER:$USER" $iso_path
      break
    fi
    sleep 3
  done
}
```

The `deploy_virtual_machine` function is designed to automate the deployment of VMs with custom settings. It supports various options for configuring the VM, including:

_Required Flags_  
* `-n`, `--name`: The name of the VM
* `-i`, `--iso`: The path to the ISO image
* `-u`, `--username`: The username for the VM
* `-p`, `--password`: The password for the VM
* `-rp`, `--root-password`: The root password for the VM

_Optional Flags_  
* `-d`, `--disk-size`: The disk size for the VM (default: 20G)
* `-r`, `--ram`: The RAM for the VM (default: 2048)
* `-c`, `--cpu`: The number of CPUs for the VM (default: 2)

_How it Works_  
- **Define Core Variables**: The function defines several variables, including the VM's name, disk size, RAM, and CPU specifications. Paths to image and machine storage directories are also specified, along with the user credentials.
- **Validation and Defaults**: It then validates the input parameters and sets default values for optional parameters also ensuring that the provided ISO image exists in the correct location before proceeding.
- **ISO Detection and OS Variant**: The function detects the OS variant from the ISO image and sets the `os_family` variable accordingly.
- **Disk and SSH Key Setup**: The function creates a disk image for the VM and generates SSH keys for secure access, and injects the SSH public key into the VM during installation.
- **Automated Cloud-Init, Preseed or Kickstart Files**: Depending on the OS variant, cloud-init files for Ubuntu, preseed configurations for Debian, or kickstart files for CentOS are generated. These configuration files automate the installation and configuration process during VM creation.
- **VM Creation Using `virt-install`**: The script builds a `virt-install` command that dynamically adds the required options for each OS. It ensures that networking, storage, and other configurations are properly set up.
- **Post-Installation Cleanup**  
After the VM installation is complete, the script waits for the VM to shut down, cleans up temporary files, and detaches the seed ISO used for installation.

