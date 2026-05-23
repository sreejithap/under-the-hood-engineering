# Automatically Installing CoreOS/RHCOS to the Smallest Drive via iPXE

## Background

I was working on automating bare metal deployments of Red Hat CoreOS (RHCOS) for OpenShift clusters using iPXE boot. My challenge was to automatically select and install the OS to the **smallest available drive** on each server, rather than hardcoding a specific device like `/dev/sda`.

This is particularly useful in heterogeneous environments where:
- Servers have different drive configurations
- Drive detection order varies
- You want to reserve larger drives for data storage
- Boot drives are consistently the smallest capacity drives

## The Challenge

My existing iPXE template hardcoded the installation device:

```ipxe
coreos.inst.install_dev=/dev/sda
```

The problem? Not all servers had their smallest drive as `/dev/sda`. Drive letter assignment (`sda`, `sdb`, `sdc`) is dynamic and depends on:
- BIOS boot order
- Drive controller enumeration
- Drive type (SATA vs NVMe vs SAS)
- Physical slot position

I needed a solution that would automatically detect and install to the smallest drive using:

```bash
lsblk -Sn -o NAME --sort SIZE | head -1
```

## Exploration of Solutions

### Option 1: Execute Shell Commands in iPXE (❌ Not Possible)

My first thought was to run the `lsblk` command directly in the iPXE template:

```ipxe
# This DOESN'T work - iPXE can't execute shell commands
coreos.inst.install_dev=$(lsblk -Sn -o NAME --sort SIZE | head -1)
```

**Result:** iPXE doesn't support shell command execution. It's a network boot firmware, not a full shell environment.

### Option 2: Wildcard Patterns (⚠️ Limited)

iPXE and CoreOS installer support device path patterns:

```ipxe
coreos.inst.install_dev=/dev/disk/by-id/scsi-*
coreos.inst.install_dev=/dev/disk/by-id/ata-*
coreos.inst.install_dev=/dev/disk/by-id/nvme-*
```

**How it works:**
- Uses persistent device identifiers (stable across reboots)
- CoreOS installer selects the first matching device alphabetically

**Limitations:**
- Does NOT guarantee the smallest drive
- Picks first alphabetically, not by size
- Only works if your smallest drives consistently match the pattern

**When to use:**
- All servers have identical hardware
- Smallest drive is always the same model/type
- You can predict which pattern matches your boot drives

### Option 3: Ignition-Based Auto-Selection (✅ Best Solution)

Since iPXE can't execute shell commands, the solution is to run the selection logic **after** the system boots into the CoreOS live environment, using Ignition configuration.

## Final Implementation

### Architecture

1. **iPXE** boots CoreOS live image WITHOUT specifying install device
2. **Ignition** provisions a systemd service that runs on first boot
3. **systemd service** executes the `lsblk` command to find the smallest drive
4. **coreos-installer** installs to the detected drive
5. **System reboots** into installed OS

### Modified iPXE Template

```ipxe
#!ipxe
 
isset ${dhcp_mac} || set dhcp_mac ${net0/mac:hexhyp}
 
<% if @host.params['mtu'] -%>
set mtu <%= @host.params['mtu'] %>
<% else -%>
set mtu 9000
<% end -%>
 
<% if @host.params['nic1'] -%>
set nic1 <%= @host.params['nic1'] %>
<% else -%>
set nic1 eno1
<% end -%>
 
<% if @host.params['nic2'] -%>
set nic2 <%= @host.params['nic2'] %>
<% else -%>
set nic2 eno2
<% end -%>
 
<% if @host.params['node_role'] -%>
set node_role <%= @host.params['node_role'] %>
<% else -%>
set node_role worker
<% end -%>
 
<% if @host.params['bac_cluster'] -%>
set bac_cluster <%= @host.params['bac_cluster'] %>
<% else -%>
set bac_cluster useast18
<% end -%>
 
<% if @host.params['ignition_server'] -%>
set ignition_server <%= @host.params['ignition_server'] %>
<% else -%>
set ignition_server generic
<% end -%>
 
<% if @host.params['lacp'] -%>
set networkargs bond=bond0:<%= @host.params['nic1'] %>,<%= @host.params['nic2'] %>:mode=802.3ad,lacp_rate=1,miimon=100,updelay=1000,downdelay=1000 ip=bond0:dhcp
<% else -%>
set networkargs bond=bond0:<%= @host.params['nic1'] %>,<%= @host.params['nic2'] %>:mode=active-backup,primary=<%= @host.params['nic1'] %>,miimon=100  ip=bond0:dhcp
<% end -%>
 
kernel -n kernel http://<%= @host.params['ignition_server'] %>:8080/ocp4/rhcos/<%= @host.params['coreos_version'] %>/rhcos-<%= @host.params['coreos_version'] %>-x86_64-live-kernel-x86_64
 
# NOTE: coreos.inst.install_dev is NOT specified - Ignition will handle installation
imgargs kernel initrd=initrd ipv6.disable=1 ${networkargs} rd.neednet=1 coreos.live.rootfs_url=http://<%= @host.params['ignition_server'] %>:8080/ocp4/rhcos/<%= @host.params['coreos_version'] %>/rhcos-<%= @host.params['coreos_version'] %>-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://<%= @host.params['ignition_server'] %>:8080/ocp4/ignition/<%= @host.params['bac_cluster'] %>/<%= @host.params['node_role'] %>.ign bootdevice=link BOOTIF=01-${dhcp_mac} foreman=<%= foreman_url(action = "built") %>
 
initrd -n initrd http://<%= @host.params['ignition_server'] %>:8080/ocp4/rhcos/<%= @host.params['coreos_version'] %>/rhcos-<%= @host.params['coreos_version'] %>-x86_64-live-initramfs.x86_64.img
 
boot
```

**Key change:** Removed `coreos.inst.install_dev=/dev/sda` from the `imgargs kernel` line.

### OpenShift Manifest for Auto-Installation

Create `manifests/99-worker-auto-drive-installer.yaml`:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-auto-drive-installer
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /usr/local/bin/install-coreos-auto.sh
          mode: 0755
          contents:
            inline: |
              #!/bin/bash
              set -euxo pipefail
              
              # Log everything
              exec > >(tee /var/log/coreos-auto-install.log)
              exec 2>&1
              
              echo "=== CoreOS Auto-Installation Script ==="
              echo "Started at: $(date)"
              
              # Check if already installed
              if [ -f /etc/coreos-install-complete ]; then
                echo "CoreOS already installed, skipping..."
                exit 0
              fi
              
              # Wait for drives to be ready
              sleep 5
              
              # Display all available drives
              echo "=== Available drives ==="
              lsblk -d
              
              # Select smallest drive using lsblk
              SMALLEST_DRIVE=$(lsblk -Sn -o NAME --sort SIZE | head -1)
              
              if [ -z "$SMALLEST_DRIVE" ]; then
                echo "ERROR: No drives found!"
                lsblk -d
                exit 1
              fi
              
              echo "=== Selected smallest drive: /dev/$SMALLEST_DRIVE ==="
              
              # Show drive details
              lsblk -o NAME,SIZE,TYPE,MODEL /dev/$SMALLEST_DRIVE
              
              # Perform installation
              echo "=== Starting CoreOS installation to /dev/$SMALLEST_DRIVE ==="
              
              coreos-installer install /dev/$SMALLEST_DRIVE \
                --ignition-url=http://IGNITION_SERVER:8080/ocp4/ignition/CLUSTER_NAME/worker.ign \
                --insecure-ignition \
                --architecture=x86_64 \
                || {
                  echo "ERROR: Installation failed!"
                  exit 1
                }
              
              echo "=== Installation completed successfully ==="
              
              # Mark as complete
              touch /etc/coreos-install-complete
              
              echo "=== Rebooting in 5 seconds ==="
              sleep 5
              systemctl reboot
    systemd:
      units:
        - name: coreos-installer-auto.service
          enabled: true
          contents: |
            [Unit]
            Description=Auto-install CoreOS to smallest drive
            Wants=network-online.target
            After=network-online.target
            ConditionPathExists=!/etc/coreos-install-complete
            
            [Service]
            Type=oneshot
            ExecStart=/usr/local/bin/install-coreos-auto.sh
            ExecStartPost=/usr/bin/touch /etc/coreos-install-complete
            StandardOutput=journal+console
            StandardError=journal+console
            TimeoutStartSec=600
            Restart=on-failure
            RestartSec=10
            
            [Install]
            WantedBy=multi-user.target
```

Create a similar file for masters: `manifests/99-master-auto-drive-installer.yaml` (same content, but change the role label to `master` and ignition URL to `master.ign`).

## Workflow Integration

### Step 1: Generate Base Manifests

```bash
# Place your install-config.yaml in the working directory
openshift-install create manifests --dir=./cluster-config
```

### Step 2: Add Custom Manifests

```bash
# Copy the auto-installer manifests
cp manifests/99-worker-auto-drive-installer.yaml ./cluster-config/manifests/
cp manifests/99-master-auto-drive-installer.yaml ./cluster-config/manifests/

# Update placeholders in the manifests
sed -i 's|IGNITION_SERVER|your-server.example.com|g' ./cluster-config/manifests/99-*.yaml
sed -i 's|CLUSTER_NAME|useast18|g' ./cluster-config/manifests/99-*.yaml
```

### Step 3: Generate Ignition Configs

```bash
# This merges all manifests into final ignition files
openshift-install create ignition-configs --dir=./cluster-config
```

### Step 4: Deploy Ignition Files

```bash
# Copy to your ignition server
scp ./cluster-config/worker.ign user@ignition-server:/var/www/html/ocp4/ignition/useast18/
scp ./cluster-config/master.ign user@ignition-server:/var/www/html/ocp4/ignition/useast18/
```

## How It Works

1. **PXE Boot**: Server boots via iPXE, loads CoreOS live kernel and initramfs
2. **Live Environment**: System boots into RAM without installing
3. **Ignition Runs**: Fetches ignition config from server
4. **Systemd Service**: `coreos-installer-auto.service` starts automatically
5. **Drive Detection**: Script runs `lsblk -Sn -o NAME --sort SIZE | head -1`
6. **Installation**: `coreos-installer` installs to detected smallest drive
7. **Reboot**: System reboots into installed OS on the smallest drive

## Debugging and Verification

### Check Drive Selection Before Installation

To see what drives are available before committing to installation, add `rd.break=pre-mount` to the iPXE kernel args temporarily:

```ipxe
imgargs kernel initrd=initrd ipv6.disable=1 ${networkargs} rd.neednet=1 rd.break=pre-mount coreos.live.rootfs_url=...
```

This drops you into an emergency shell where you can run:

```bash
# List all drives
lsblk -d

# Show drives sorted by size
lsblk -Sn -o NAME,SIZE,MODEL --sort SIZE

# Test the selection command
lsblk -Sn -o NAME --sort SIZE | head -1

# View persistent identifiers
ls -l /dev/disk/by-id/
ls -l /dev/disk/by-path/

# Exit to continue boot
exit
```

### View Installation Logs

After installation completes, you can check the logs on the installed system:

```bash
# View the auto-install log
journalctl -u coreos-installer-auto.service

# Or check the log file directly
cat /var/log/coreos-auto-install.log
```

## Alternative Approach: Pattern-Based Selection

If your environment has predictable hardware configurations, you can use the simpler pattern-based approach in iPXE:

```ipxe
# For SCSI/SATA drives
coreos.inst.install_dev=/dev/disk/by-id/scsi-*

# For NVMe drives
coreos.inst.install_dev=/dev/disk/by-id/nvme-*

# For ATA drives
coreos.inst.install_dev=/dev/disk/by-id/ata-*
```

**Pros:**
- No Ignition modification needed
- Simple configuration
- Uses stable device identifiers

**Cons:**
- Does NOT guarantee smallest drive
- Picks first match alphabetically
- Only works with homogeneous hardware

## Lessons Learned

1. **iPXE Limitations**: iPXE is a boot firmware, not a shell environment. It cannot execute shell commands or dynamic scripting.
2. **Device Naming is Dynamic**: `/dev/sdX` letters change between boots and systems. Always use persistent identifiers (`/dev/disk/by-id/`, `/dev/disk/by-path/`) for production.
3. **Ignition is Powerful**: CoreOS Ignition configs can provision complete systemd services, scripts, and configuration files during first boot.
4. **Two-Phase Installation**: CoreOS supports a two-phase installation:
   - Phase 1: Boot live environment with Ignition
   - Phase 2: Ignition triggers `coreos-installer` to install to disk
5. **Manifest Injection**: OpenShift's `openshift-install` workflow supports injecting custom manifests between the `create manifests` and `create ignition-configs` steps.

## Conclusion

While iPXE cannot directly execute shell commands to detect the smallest drive, combining iPXE boot with Ignition-based automation provides a robust solution. The key insight is letting the live CoreOS environment handle drive detection and installation, rather than trying to do it at the iPXE level.

This approach works reliably across heterogeneous hardware and ensures the OS is always installed to the smallest available drive, regardless of drive enumeration order or device naming.

## References

- [CoreOS Installer Documentation](https://coreos.github.io/coreos-installer/)
- [Fedora CoreOS Ignition Configuration](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/)
- [OpenShift Installation Customization](https://docs.openshift.com/container-platform/latest/installing/install_config/installing-customizing.html)
- [iPXE Command Reference](https://ipxe.org/cmd)
- [lsblk Manual Page](https://man7.org/linux/man-pages/man8/lsblk.8.html)
