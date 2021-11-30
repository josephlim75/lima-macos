[Lima](https://github.com/lima-vm/lima) (Linux virtual machines, on macOS) installation guide for M1 Mac.

**Sep. 27th 2021 UPDATED**

Now we can install patched version of QEMU via Homebrew (thank you everyone for the info!). Here is the updated instruction with it:

Used M1 Mac mini 2020 with macOS Big Sur Version 11.6.

## 1. Install QEMU & Lima

### 1-1. Uninstall existing (manually installed) QEMU and Lima 

**_Note: It is safe to remove them before proceeding if you already have manually installed "patched QEMU" and "Lima" in your M1 Mac. This document assumes you've installed it by following this [guide](https://gist.github.com/toricls/d3dd0bec7d4c6ddbcf2d25f211e8cd7b#1-install-patched-qemu)._**

```shell
# !! CAUTION !!
# This document assumes the below paths that you've installed by following https://gist.github.com/toricls/d3dd0bec7d4c6ddbcf2d25f211e8cd7b#1-install-patched-qemu
$ sudo rm -f /usr/local/bin/{lima,limactl,nerdctl}
$ sudo rm -rf /usr/local/share/lima
$ sudo rm -f /usr/local/bin/{qemu-edid,qemu-ga,qemu-img,qemu-io,qemu-nbd,qemu-storage-daemon,qemu-system-aarch64}
```

### 1-2. Install QEMU and Lima using Homebrew

```shell
# Update Homebrew formulae
$ brew update

# Install QEMU
$ brew install qemu

# Check version
$ which qemu-system-aarch64
/opt/homebrew/bin/qemu-system-aarch64

$ qemu-system-aarch64 --version
QEMU emulator version 6.1.0
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers

# Check if its valid
$ qemu-system-aarch64 -accel help
Accelerators supported in QEMU binary:
hvf
tcg

# Install Lima
$ brew install lima

# Check version
$ which limactl
/opt/homebrew/bin/limactl

$ limactl --version
limactl version 0.6.4
```

## 2. Launch a Lima VM and run a test container

```shell
# Remove existing VMs (if any) for safe
$ limactl list
```

Create and move to working directory

```
$ mkdir -p /tmp/lima/qemu-lima-via-homebrew && cd $_
```

Save the following as `default.yaml` in `/tmp/lima/qemu-lima-via-homebrew`. It's altered from the [original file](https://github.com/lima-vm/lima/blob/a4920c1907fa3028689962a8abe29d2ea0f24e9a/pkg/limayaml/default.yaml) for 1) setting `loadDotSSHPubKeys` to false, and 2) changing num of cpu and disk size.

```yaml
# ===================================================================== #
# BASIC CONFIGURATION
# ===================================================================== #

# Arch: "default", "x86_64", "aarch64".
# "default" corresponds to the host architecture.
arch: "default"

# An image must support systemd and cloud-init.
# Ubuntu and Fedora are known to work.
# Default: none (must be specified)
images:
  # Try to use a local image first.
  - location: "~/Downloads/hirsute-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "~/Downloads/hirsute-server-cloudimg-arm64.img"
    arch: "aarch64"

  # Download the file from the internet when the local file is missing.
  # Hint: run `limactl prune` to invalidate the "current" cache
  - location: "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-arm64.img"
    arch: "aarch64"

# CPUs: if you see performance issues, try limiting cpus to 1.
# Default: 4
cpus: 2

# Memory size
# Default: "4GiB"
memory: "4GiB"

# Disk size
# Default: "100GiB"
disk: "30GiB"

# Expose host directories to the guest, the mount point might be accessible from all UIDs in the guest
# Default: none
mounts:
  - location: "~"
    # CAUTION: `writable` SHOULD be false for the home directory.
    # Setting `writable` to true is possible, but untested and dangerous.
    writable: false
  - location: "/tmp/lima"
    writable: true

ssh:
  # A localhost port of the host. Forwarded to port 22 of the guest.
  # Currently, this port number has to be specified manually.
  # Default: none
  localPort: 60022
  # Load ~/.ssh/*.pub in addition to $LIMA_HOME/_config/user.pub .
  # This option is useful when you want to use other SSH-based
  # applications such as rsync with the Lima instance.
  # If you have an insecure key under ~/.ssh, do not use this option.
  # Default: true
  loadDotSSHPubKeys: false

# ===================================================================== #
# ADVANCED CONFIGURATION
# ===================================================================== #

containerd:
  # Enable system-wide (aka rootful)  containerd and its dependencies (BuildKit, Stargz Snapshotter)
  # Default: false
  system: false
  # Enable user-scoped (aka rootless) containerd and its dependencies
  # Default: true
  user: true

# Provisioning scripts need to be idempotent because they might be called
# multiple times, e.g. when the host VM is being restarted.
# provision:
#   # `system` is executed with the root privilege
#   - mode: system
#     script: |
#       #!/bin/bash
#       set -eux -o pipefail
#       export DEBIAN_FRONTEND=noninteractive
#       apt-get install -y vim
#   # `user` is executed without the root privilege
#   - mode: user
#     script: |
#       #!/bin/bash
#       set -eux -o pipefail
#       cat <<EOF > ~/.vimrc
#       set number
#       EOF

# probes:
#  # Only `readiness` probes are supported right now.
#  - mode: readiness
#    description: vim to be installed
#    script: |
#       #!/bin/bash
#       set -eux -o pipefail
#       if ! timeout 30s bash -c "until command -v vim; do sleep 3; done"; then
#         echo >&2 "vim is not installed yet"
#         exit 1
#       fi
#    hint: |
#      vim was not installed in the guest. Make sure the package system is working correctly.
#      Also see "/var/log/cloud-init-output.log" in the guest.

# ===================================================================== #
# FURTHER ADVANCED CONFIGURATION
# ===================================================================== #

firmware:
  # Use legacy BIOS instead of UEFI.
  # Default: false
  legacyBIOS: false

video:
  # QEMU display, e.g., "none", "cocoa", "sdl".
  # As of QEMU v5.2, enabling this is known to have negative impact
  # on performance on macOS hosts: https://gitlab.com/qemu-project/qemu/-/issues/334
  # Default: "none"
  display: "none"

# The instance can get routable IP addresses from the vmnet framework using
# https://github.com/lima-vm/vde_vmnet.
networks:
  # Lima can manage daemons for networks defined in $LIMA_HOME/_config/networks.yaml
  # automatically. Both vde_switch and vde_vmnet binaries must be installed into
  # secure locations only alterable by the "root" user.
  # - lima: shared
  #   # MAC address of the instance; lima will pick one based on the instance name,
  #   # so DHCP assigned ip addresses should remain constant over instance restarts.
  #   macAddress: ""
  #   # Interface name, defaults to "lima0", "lima1", etc.
  #   interface: ""
  #
  # Lima can also connect to "unmanaged" vde networks addressed by "vnl". This
  # means that the daemons will not be controlled by Lima, but must be started
  # before the instance.  The interface type (host, shared, or bridged) is
  # configured in vde_vmnet and not in lima.
  # vnl (virtual network locator) points to the vde_switch socket directory,
  # optionally with vde:// prefix
  # - vnl: "vde:///var/run/vde.ctl"
  #   # VDE Switch port number (not TCP/UDP port number). Set to 65535 for PTP mode.
  #   # Default: 0
  #   switchPort: 0
  #   # MAC address of the instance; lima will pick one based on the instance name,
  #   # so DHCP assigned ip addresses should remain constant over instance restarts.
  #   macAddress: ""
  #   # Interface name, defaults to "lima0", "lima1", etc.
  #   interface: ""

# Port forwarding rules. Forwarding between ports 22 and ssh.localPort cannot be overridden.
# Rules are checked sequentially until the first one matches.
# portForwards:
#   - guestPort: 443
#     hostIP: "0.0.0.0" # overrides the default value "127.0.0.1"; allows privileged port forwarding
#   # default: hostPort: 443 (same as guestPort)
#   # default: guestIP: "127.0.0.1" (also matches bind addresses "0.0.0.0", "::", and "::1")
#   # default: proto: "tcp" (only valid value right now)
#   - guestPortRange: [4000, 4999]
#     hostIP:  "0.0.0.0" # overrides the default value "127.0.0.1"
#   # default: hostPortRange: [4000, 4999] (must specify same number of ports as guestPortRange)
#   - guestPort: 80
#     hostPort: 8080 # overrides the default value 80
#   - guestIP: "127.0.0.2" # overrides the default value "127.0.0.1"
#     hostIP: "127.0.0.2" # overrides the default value "127.0.0.1"
#   # default: guestPortRange: [1024, 65535]
#   # default: hostPortRange: [1024, 65535]
#   - guestPort: 8888
#     ignore: true (don't forward this port)
#   # Lima internally appends this fallback rule at the end:
#   - guestIP: "127.0.0.1"
#     guestPortRange: [1024, 65535]
#     hostIP: "127.0.0.1"
#     hostPortRange: [1024, 65535]
#   # Any port still not matched by a rule will not be forwarded (ignored)

# Extra environment variables that will be loaded into the VM at start up.
# These variables are currently only consumed by internal init scripts, not by the user shell.
# This field is experimental and may change in a future release of Lima.
# https://github.com/lima-vm/lima/pull/200
# env:
#   KEY: value

# Explicitly set DNS addresses for qemu user-mode networking. By default qemu picks *one*
# nameserver from the host config and forwards all queries to this server. On macOS
# Lima adds the nameservers configured for the "en0" interface to the list. In case this
# still doesn't work (e.g. VPN setups), the servers can be specified here explicitly.
# If nameservers are specified here, then the "en0" configuration will be ignored.
# dns:
# - 1.1.1.1
# - 1.0.0.1

# ===================================================================== #
# END OF TEMPLATE
# ===================================================================== #
```

### 2-2. Start a VM

```shell
$ pwd
/tmp/lima/qemu-lima-via-homebrew

$ limactl start default.yaml
? Creating an instance "default" Proceed with the default configuration
INFO[0021] Downloading "https://github.com/containerd/nerdctl/releases/download/v0.11.2/nerdctl-full-0.11.2-linux-arm64.tar.gz" (sha256:fe6322a88cb15d8a502e649827e3d1570210bb038b7a4a52820bce0fec86a637)
154.29 MiB / 154.29 MiB [----------------------------------] 100.00% 25.94 MiB/s
INFO[0030] Downloaded "nerdctl-full-0.11.2-linux-arm64.tar.gz"
INFO[0030] Attempting to download the image from "~/Downloads/hirsute-server-cloudimg-arm64.img"
INFO[0030] Attempting to download the image from "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-arm64.img"
INFO[0031] Using cache "/Users/xxxx/Library/Caches/lima/download/by-url-sha256/c40afd2acd5f2078759e01e119168213674615fff0701d53bc2bf69e5be3bf69/data"
INFO[0032] [hostagent] Starting QEMU (hint: to watch the boot progress, see "/Users/xxxx/.lima/default/serial.log")
INFO[0032] SSH Local Port: 60022
INFO[0032] [hostagent] Waiting for the essential requirement 1 of 4: "ssh"
INFO[0042] [hostagent] Waiting for the essential requirement 1 of 4: "ssh"
INFO[0049] [hostagent] The essential requirement 1 of 4 is satisfied
INFO[0049] [hostagent] Waiting for the essential requirement 2 of 4: "sshfs binary to be installed"
INFO[0061] [hostagent] The essential requirement 2 of 4 is satisfied
INFO[0061] [hostagent] Waiting for the essential requirement 3 of 4: "/etc/fuse.conf to contain \"user_allow_other\""
INFO[0070] [hostagent] The essential requirement 3 of 4 is satisfied
INFO[0070] [hostagent] Waiting for the essential requirement 4 of 4: "the guest agent to be running"
INFO[0070] [hostagent] The essential requirement 4 of 4 is satisfied
INFO[0070] [hostagent] Mounting "/Users/xxxx"
INFO[0070] [hostagent] Mounting "/tmp/lima"
INFO[0070] [hostagent] Waiting for the optional requirement 1 of 2: "systemd must be available"
INFO[0070] [hostagent] Forwarding "/run/user/505/lima-guestagent.sock" (guest) to "/Users/xxxx/.lima/default/ga.sock" (host)
INFO[0071] [hostagent] Not forwarding TCP [::]:22
INFO[0071] [hostagent] The optional requirement 1 of 2 is satisfied
INFO[0071] [hostagent] Waiting for the optional requirement 2 of 2: "containerd binaries to be installed"
INFO[0071] [hostagent] Not forwarding TCP 127.0.0.53:53
INFO[0071] [hostagent] Not forwarding TCP 0.0.0.0:22
INFO[0074] [hostagent] The optional requirement 2 of 2 is satisfied
INFO[0074] READY. Run `lima` to open the shell.
```

### 2-3. Run an nginx container inside the Lima VM

```shell
$ lima nerdctl run -d --name nginx -p 127.0.0.1:8080:80 nginx:alpine
docker.io/library/nginx:alpine:                                                   resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:686aac2769fd6e7bab67663fd38750c135b72d993d0bb0a942ab02ef647fc9c3:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:fa27d916cd6d3f1af3059dfb02cc5ce2a148728c7834f0ca16f5cca72851ba3e: done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:2e723ef9408f151d4033d6669c76aaee5daa4653f1a135970cbeb5e0dde43d28:   done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:ee8589645f052a7fecf4ff6311e5ef36367b722a10c4442aeaa43c6609db5b16:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:70156bbc617dc94210d0f96d1bf5ad0b1e263dd0ad81c76133389f62cc64f692:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:552d1f2373af9bfe12033568ebbfb0ccbb0de11279f9a415a29207e264d7f4d9:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:4d04d94e9d5d1d4a9d47c8026344e8f4c5e4295c9aef1d0198f7147305a43607:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:27d5c959473e974c9c03971010537ef645f7999d88c0266085201a9739abe4d2:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:4a4b19d41c9f7829772b7f64d968f34a9041fa728fca5d9f681d3f2a4c70e011:    done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 4.6 s                                                                    total:  9.3 Mi (2.0 MiB/s)
2ca078aaa408452bec3637c9ea42494a78c1750ba2263021b370ab2a14a945f6

$ lima nerdctl ps
CONTAINER ID    IMAGE                             COMMAND                   CREATED           STATUS    PORTS                     NAMES
2ca078aaa408    docker.io/library/nginx:alpine    "/docker-entrypoint.…"    16 seconds ago    Up        127.0.0.1:8080->80/tcp    nginx

$ curl -I http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.21.3
Date: Mon, 27 Sep 2021 02:19:28 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 07 Sep 2021 15:50:09 GMT
Connection: keep-alive
ETag: "61378a31-267"
Accept-Ranges: bytes
```

### 2-4. Use an alias `docker` for `lima nerdctl`

```shell
$ alias docker="lima nerdctl"

$ docker ps
CONTAINER ID    IMAGE                             COMMAND                   CREATED           STATUS    PORTS                     NAMES
2ca078aaa408    docker.io/library/nginx:alpine    "/docker-entrypoint.…"    59 seconds ago    Up        127.0.0.1:8080->80/tcp    nginx
```

Woot! Woot!

---

**_The below was written on Sep. 1st 2021 when we needed to patch and build QEMU by ourselves, and is now completely outdated. Keeping this just for the record._**

Used M1 Mac mini 2020 with macOS Big Sur Version 11.5.2.

# 1. Install Patched QEMU

According to the [GitHub issue comment](https://github.com/lima-vm/lima/issues/42#issuecomment-866481653) in `Lima`'s GitHub repository, Lima requires a patched QEMU on M1 Macs.

Note that the followings are customized steps by @toricls based on the [original steps](https://gist.github.com/nrjdalal/e70249bb5d2e9d844cc203fd11f74c55) and the [script](https://github.com/nrjdalal/silicon-virtualizer/blob/66bbfde663b2ab833e9724861bdec7da99470b95/install-qemu.sh). Be sure to check the original files when you try it on your own.

## Install QEMU on Silicon based Apple Macs

```shell
# Install necessary packages for building
brew install libffi gettext glib pkg-config autoconf automake pixman ninja

# Clone qemu using ghq instead of git
ghq get https://github.com/qemu/qemu

# Change directory to qemu repository
# Note: In my case, git repositories are stored under $HOME/go/src directory
cd $HOME/go/src/$(ghq list | grep qemu)

# Checkout to commit dated June 03, 2021 v6.0.0
git checkout 3c93dfa42c394fdd55684f2fbf24cf2f39b97d47

# Apply patch series v8 by Alexander Graf
curl https://patchwork.kernel.org/series/485309/mbox/ | git am

# Building qemu installer
mkdir build && cd build
../configure --target-list=aarch64-softmmu
make -j8

# Install qemu
sudo make install
```

## Create Ubuntu Server using QEMU on Silicon based Apple Macs

# "First run, close manually after installation aka first reboot." steps

```shell
# Change directory to qemu repository
# Note: Git repositories are stored under $HOME/go/src directory in my case
cd $HOME/go/src/$(ghq list | grep qemu)

mkdir /tmp/test-patched-qemu && cd /tmp/test-patched-qemu

curl https://cdimage.ubuntu.com/releases/20.04/release/ubuntu-20.04.2-live-server-arm64.iso -o ubuntu-lts.iso

qemu-img create -f qcow2 virtual-disk.qcow2 8G

cp $(dirname $(which qemu-img))/../share/qemu/edk2-aarch64-code.fd .

dd if=/dev/zero conv=sync bs=1m count=64 of=ovmf_vars.fd

# you'll see four files in the /tmp/test-patched-qemu directory, 1) edk2-aarch64-code.fd, 2) ovmf_vars.fd, 3) ubuntu-lts.iso, and 4) virtual-disk.qcow2
ls

qemu-system-aarch64 \
  -machine virt,accel=hvf,highmem=off \
  -cpu cortex-a72 -smp 4 -m 4G \
  -device virtio-gpu-pci \
  -device virtio-keyboard-pci \
  -drive "format=raw,file=edk2-aarch64-code.fd,if=pflash,readonly=on" \
  -drive "format=raw,file=ovmf_vars.fd,if=pflash" \
  -drive "format=qcow2,file=virtual-disk.qcow2" \
  -cdrom ubuntu-lts.iso

# I just tested just starting up it without any issues and skipped the "Second run, enjoy your Ubuntu Server." step, but you can of course finish instalation steps youself for your newly created VM to interact with it
```

# 2. Play with Lima

## Installation

```shell
# According to the [installation doc](https://github.com/lima-vm/lima#installation), Lima supports Homebrew only for Intel Macs.

# Installing Lima manually
mkdir /tmp/lima && cd /tmp/lima

curl -L https://github.com/lima-vm/lima/releases/download/v0.6.1/lima-0.6.1-Darwin-arm64.tar.gz -o ./lima-0.6.1-Darwin-arm64.tar.gz

# List content in the tar.gz file
tar tzf /tmp/lima/lima-0.6.1-Darwin-arm64.tar.gz

# Extract them
tar -xzvf /tmp/lima/lima-0.6.1-Darwin-arm64.tar.gz -C /tmp/lima

# Rename the command "nerdctl.lima" to "nerdctl"
mv /tmp/lima/bin/nerdctl.lima /tmp/lima/bin/nerdctl

# Move the commands
sudo mv ./bin/* /usr/local/bin/
sudo mv ./share/lima /usr/local/share
```

## Start VM

Save the following as `default.yaml`. It's altered from the [original file](https://github.com/lima-vm/lima/blob/master/pkg/limayaml/default.yaml) for 1) setting `loadDotSSHPubKeys` to false, 2) removing the "mounts" section, and 3) changing num of cpu and disk size.

```
# ===================================================================== #
# BASIC CONFIGURATION
# ===================================================================== #

# Arch: "default", "x86_64", "aarch64".
# "default" corresponds to the host architecture.
arch: "default"

# An image must support systemd and cloud-init.
# Ubuntu and Fedora are known to work.
# Default: none (must be specified)
images:
  # Try to use a local image first.
  - location: "~/Downloads/hirsute-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "~/Downloads/hirsute-server-cloudimg-arm64.img"
    arch: "aarch64"

  # Download the file from the internet when the local file is missing.
  # Hint: run `limactl prune` to invalidate the "current" cache
  - location: "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-arm64.img"
    arch: "aarch64"

# CPUs: if you see performance issues, try limiting cpus to 1.
# Default: 4
cpus: 2

# Memory size
# Default: "4GiB"
memory: "4GiB"

# Disk size
# Default: "100GiB"
disk: "30GiB"

ssh:
  # A localhost port of the host. Forwarded to port 22 of the guest.
  # Currently, this port number has to be specified manually.
  # Default: none
  localPort: 60022
  # Load ~/.ssh/*.pub in addition to $LIMA_HOME/_config/user.pub .
  # This option is useful when you want to use other SSH-based
  # applications such as rsync with the Lima instance.
  # If you have an insecure key under ~/.ssh, do not use this option.
  # Default: true
  loadDotSSHPubKeys: false

# ===================================================================== #
# ADVANCED CONFIGURATION
# ===================================================================== #

containerd:
  # Enable system-wide (aka rootful)  containerd and its dependencies (BuildKit, Stargz Snapshotter)
  # Default: false
  system: false
  # Enable user-scoped (aka rootless) containerd and its dependencies
  # Default: true
  user: true

# ===================================================================== #
# FURTHER ADVANCED CONFIGURATION
# ===================================================================== #

firmware:
  # Use legacy BIOS instead of UEFI.
  # Default: false
  legacyBIOS: false

video:
  # QEMU display, e.g., "none", "cocoa", "sdl".
  # As of QEMU v5.2, enabling this is known to have negative impact
  # on performance on macOS hosts: https://gitlab.com/qemu-project/qemu/-/issues/334
  # Default: "none"
  display: "none"

network:
  # The instance can get routable IP addresses from the vmnet framework using
  # https://github.com/lima-vm/vde_vmnet. Both vde_switch and vde_vmnet
  # daemons must be running before the instance is started. The interface type
  # (host, shared, or bridged) is configured in vde_vmnet and not lima.
  vde:
    # vnl (virtual network locator) points to the vde_switch socket directory,
    # optionally with vde:// prefix
    # - vnl: "vde:///var/run/vde.ctl"
    #   # VDE Switch port number (not TCP/UDP port number). Set to 65535 for PTP mode.
    #   # Default: 0
    #   switchPort: 0
    #   # MAC address of the instance; lima will pick one based on the instance name,
    #   # so DHCP assigned ip addresses should remain constant over instance restarts.
    #   macAddress: ""
    #   # Interface name, defaults to "vde0", "vde1", etc.
    #   name: ""

# Port forwarding rules. Forwarding between ports 22 and ssh.localPort cannot be overridden.
# Rules are checked sequentially until the first one matches.
# portForwards:
#   - guestPort: 443
#     hostIP: "0.0.0.0" # overrides the default value "127.0.0.1"; allows privileged port forwarding
#   # default: hostPort: 443 (same as guestPort)
#   # default: guestIP: "127.0.0.1" (also matches bind addresses "0.0.0.0", "::", and "::1")
#   # default: proto: "tcp" (only valid value right now)
#   - guestPortRange: [4000, 4999]
#     hostIP:  "0.0.0.0" # overrides the default value "127.0.0.1"
#   # default: hostPortRange: [4000, 4999] (must specify same number of ports as guestPortRange)
#   - guestPort: 80
#     hostPort: 8080 # overrides the default value 80
#   - guestIP: "127.0.0.2" # overrides the default value "127.0.0.1"
#     hostIP: "127.0.0.2" # overrides the default value "127.0.0.1"
#   # default: guestPortRange: [1024, 65535]
#   # default: hostPortRange: [1024, 65535]
#   - guestPort: 8888
#     ignore: true (don't forward this port)
#   # Lima internally appends this fallback rule at the end:
#   - guestIP: "127.0.0.1"
#     guestPortRange: [1024, 65535]
#     hostIP: "127.0.0.1"
#     hostPortRange: [1024, 65535]
#   # Any port still not matched by a rule will not be forwarded (ignored)

# ===================================================================== #
# END OF TEMPLATE
# ===================================================================== #
```

```
$ limactl start default.yaml
? Creating an instance "default" Proceed with the default configuration
INFO[0001] Downloading "https://github.com/containerd/nerdctl/releases/download/v0.11.1/nerdctl-full-0.11.1-linux-arm64.tar.gz" (sha256:e2c8d0417b2fb79919f22a818813c646ad7ce0e600a951b6bac98340650e4435)
INFO[0001] Using cache "/Users/xxxx/Library/Caches/lima/download/by-url-sha256/cf356d64aa826ad177396feef432a14e0df5b2bb998191eff281541d329ff12c/data"
INFO[0001] Attempting to download the image from "~/Downloads/hirsute-server-cloudimg-arm64.img"
INFO[0001] Attempting to download the image from "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-arm64.img"
INFO[0002] Using cache "/Users/xxxx/Library/Caches/lima/download/by-url-sha256/c40afd2acd5f2078759e01e119168213674615fff0701d53bc2bf69e5be3bf69/data"
INFO[0002] [hostagent] Starting QEMU (hint: to watch the boot progress, see "/Users/xxxx/.lima/default/serial.log")
INFO[0002] SSH Local Port: 60022
INFO[0002] [hostagent] Waiting for the essential requirement 1 of 2: "ssh"
INFO[0012] [hostagent] Waiting for the essential requirement 1 of 2: "ssh"
INFO[0019] [hostagent] The essential requirement 1 of 2 is satisfied
INFO[0019] [hostagent] Waiting for the essential requirement 2 of 2: "the guest agent to be running"
INFO[0022] [hostagent] The essential requirement 2 of 2 is satisfied
INFO[0022] [hostagent] Waiting for the optional requirement 1 of 2: "systemd must be available"
INFO[0022] [hostagent] Forwarding "/run/user/505/lima-guestagent.sock" (guest) to "/Users/xxxx/.lima/default/ga.sock" (host)
INFO[0022] [hostagent] The optional requirement 1 of 2 is satisfied
INFO[0022] [hostagent] Waiting for the optional requirement 2 of 2: "containerd binaries to be installed"
INFO[0022] [hostagent] Not forwarding TCP 127.0.0.53:53
INFO[0022] [hostagent] Not forwarding TCP 0.0.0.0:22
INFO[0022] [hostagent] Not forwarding TCP [::]:22
INFO[0037] [hostagent] The optional requirement 2 of 2 is satisfied
INFO[0037] READY. Run `lima` to open the shell.
```

## Run nginx container inside Lima VM

```
$ lima nerdctl run -d --name nginx -p 127.0.0.1:8080:80 nginx:alpine

docker.io/library/nginx:alpine:                                                   resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:859ec6f2dc548cd2e5144b7856f2b5c37b23bd061c0c93cfa41fb5fb78307ead:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:4375f141ad62ca2763d3698e81ab6c61fa2cb8dba169212bf92c845158dbafb1: done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:78754800625ea92fe7b32be0754194394e84a2ec0018b037f4279822bfcf5712:   done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:0a8cd3c047c9c865bf65fa310e119ccbd3264a4050c81d0d6b5c3fd153dcce30:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:552d1f2373af9bfe12033568ebbfb0ccbb0de11279f9a415a29207e264d7f4d9:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:50f60b67a614a5a3025f6d389391af19154139b35d314b2aa89dcd2791c4050b:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:e14bcbebf464b5696ca521435ef24348933f399f49bc4c3f4bce4a6c9b6fce5b:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:d8e1f9d1b87e92444b4e8e5dab10322aa2d66c1a226e1419048328a36fc0f716:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:4d5947aec77d85cbbdc91c6bf3435095c91d516cf90af8d1394d1538d0103dcd:    done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 5.1 s                                                                    total:  9.3 Mi (1.8 MiB/s)
ab3ff970790da2ca30a7007b80e1fb65bbe86a4e39809096f764ec8c4cdbdba7

$ curl -I http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.21.1
Date: Wed, 01 Sep 2021 04:23:20 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 06 Jul 2021 15:21:34 GMT
Connection: keep-alive
ETag: "60e474fe-264"
Accept-Ranges: bytes
```

## Use alias `docker` for `lima nerdctl`

```
$ lima nerdctl ps
CONTAINER ID    IMAGE                             COMMAND                   CREATED           STATUS    PORTS                     NAMES
ab3ff970790d    docker.io/library/nginx:alpine    "/docker-entrypoint.…"    39 seconds ago    Up        127.0.0.1:8080->80/tcp    nginx

$ alias docker="lima nerdctl"

$ docker ps
CONTAINER ID    IMAGE                             COMMAND                   CREATED               STATUS    PORTS                     NAMES
ab3ff970790d    docker.io/library/nginx:alpine    "/docker-entrypoint.…"    About a minute ago    Up        127.0.0.1:8080->80/tcp    nginx
```

Woot! Woot!

# 3. Cont. Play with Lima

Since there was a [bind mount related bug](https://github.com/containerd/nerdctl/issues/338) in Lima 0.6.1 at the time of playing with Lima above, here are another steps to upgrade Lima 0.6.1 to 0.6.3 and to test the fixed version.

## Upgrade from Lima 0.6.1 to 0.6.3

We haven't took the homebrew path to install Lima 0.6.1 since the [doc said it's not supported on Arm Mac](https://github.com/lima-vm/lima/tree/be682789f45b2af8c5d5386920d4db091b4ab767#installation), so let's upgrade Lima by downloading the latest binaries from GitHub. 

```shell
# Installing Lima manually
rm -f /tmp/lima && mkdir /tmp/lima && cd /tmp/lima

curl -L https://github.com/lima-vm/lima/releases/download/v0.6.3/lima-0.6.3-Darwin-arm64.tar.gz -o ./lima-0.6.3-Darwin-arm64.tar.gz

# List content in the tar.gz file
tar tzf /tmp/lima/lima-0.6.3-Darwin-arm64.tar.gz

# Extract them
tar -xzvf /tmp/lima/lima-0.6.3-Darwin-arm64.tar.gz -C /tmp/lima

# Rename the command "nerdctl.lima" to "nerdctl"
mv /tmp/lima/bin/nerdctl.lima /tmp/lima/bin/nerdctl

# Remove the old binaries
sudo rm -rf /usr/local/share/lima

# Move the commands
sudo mv ./bin/* /usr/local/bin/
sudo mv ./share/lima /usr/local/share

# Make sure we have the expected version binary
lima --version
limactl version 0.6.3
```

## Recreate and start Lima VM

```shell
mkdir $HOME/lima-test-directory && cd $_
```

Save the following as `default.yaml`. It's altered from the [original file](https://github.com/lima-vm/lima/blob/master/pkg/limayaml/default.yaml) for 1) setting `loadDotSSHPubKeys` to false, and 2) changing num of cpu and disk size.

```shell
# ===================================================================== #
# BASIC CONFIGURATION
# ===================================================================== #

# Arch: "default", "x86_64", "aarch64".
# "default" corresponds to the host architecture.
arch: "default"

# An image must support systemd and cloud-init.
# Ubuntu and Fedora are known to work.
# Default: none (must be specified)
images:
  # Try to use a local image first.
  - location: "~/Downloads/hirsute-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "~/Downloads/hirsute-server-cloudimg-arm64.img"
    arch: "aarch64"

  # Download the file from the internet when the local file is missing.
  # Hint: run `limactl prune` to invalidate the "current" cache
  - location: "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-arm64.img"
    arch: "aarch64"

# CPUs: if you see performance issues, try limiting cpus to 1.
# Default: 4
# @toricls - set "2" instead of "4"
cpus: 2

# Memory size
# Default: "4GiB"
memory: "4GiB"

# Disk size
# Default: "100GiB"
# @toricls - set "30GiB" instead of "100GiB"
disk: "30GiB"

# Expose host directories to the guest
# Default: none
mounts:
  - location: "~"
    # CAUTION: `writable` SHOULD be false for the home directory.
    # Setting `writable` to true is possible, but untested and dangerous.
    writable: false
  - location: "/tmp/lima"
    writable: true

ssh:
  # A localhost port of the host. Forwarded to port 22 of the guest.
  # Currently, this port number has to be specified manually.
  # Default: none
  localPort: 60022
  # Load ~/.ssh/*.pub in addition to $LIMA_HOME/_config/user.pub .
  # This option is useful when you want to use other SSH-based
  # applications such as rsync with the Lima instance.
  # If you have an insecure key under ~/.ssh, do not use this option.
  # Default: true
  # @toricls - set "false" instead of "true"
  loadDotSSHPubKeys: false

# ===================================================================== #
# ADVANCED CONFIGURATION
# ===================================================================== #

containerd:
  # Enable system-wide (aka rootful)  containerd and its dependencies (BuildKit, Stargz Snapshotter)
  # Default: false
  system: false
  # Enable user-scoped (aka rootless) containerd and its dependencies
  # Default: true
  user: true

# ===================================================================== #
# FURTHER ADVANCED CONFIGURATION
# ===================================================================== #

firmware:
  # Use legacy BIOS instead of UEFI.
  # Default: false
  legacyBIOS: false

video:
  # QEMU display, e.g., "none", "cocoa", "sdl".
  # As of QEMU v5.2, enabling this is known to have negative impact
  # on performance on macOS hosts: https://gitlab.com/qemu-project/qemu/-/issues/334
  # Default: "none"
  display: "none"

network:
  # The instance can get routable IP addresses from the vmnet framework using
  # https://github.com/lima-vm/vde_vmnet. Both vde_switch and vde_vmnet
  # daemons must be running before the instance is started. The interface type
  # (host, shared, or bridged) is configured in vde_vmnet and not lima.
  vde:
    # vnl (virtual network locator) points to the vde_switch socket directory,
    # optionally with vde:// prefix
    # - vnl: "vde:///var/run/vde.ctl"
    #   # VDE Switch port number (not TCP/UDP port number). Set to 65535 for PTP mode.
    #   # Default: 0
    #   switchPort: 0
    #   # MAC address of the instance; lima will pick one based on the instance name,
    #   # so DHCP assigned ip addresses should remain constant over instance restarts.
    #   macAddress: ""
    #   # Interface name, defaults to "vde0", "vde1", etc.
    #   name: ""

# Port forwarding rules. Forwarding between ports 22 and ssh.localPort cannot be overridden.
# Rules are checked sequentially until the first one matches.
# portForwards:
#   - guestPort: 443
#     hostIP: "0.0.0.0" # overrides the default value "127.0.0.1"; allows privileged port forwarding
#   # default: hostPort: 443 (same as guestPort)
#   # default: guestIP: "127.0.0.1" (also matches bind addresses "0.0.0.0", "::", and "::1")
#   # default: proto: "tcp" (only valid value right now)
#   - guestPortRange: [4000, 4999]
#     hostIP:  "0.0.0.0" # overrides the default value "127.0.0.1"
#   # default: hostPortRange: [4000, 4999] (must specify same number of ports as guestPortRange)
#   - guestPort: 80
#     hostPort: 8080 # overrides the default value 80
#   - guestIP: "127.0.0.2" # overrides the default value "127.0.0.1"
#     hostIP: "127.0.0.2" # overrides the default value "127.0.0.1"
#   # default: guestPortRange: [1024, 65535]
#   # default: hostPortRange: [1024, 65535]
#   - guestPort: 8888
#     ignore: true (don't forward this port)
#   # Lima internally appends this fallback rule at the end:
#   - guestIP: "127.0.0.1"
#     guestPortRange: [1024, 65535]
#     hostIP: "127.0.0.1"
#     hostPortRange: [1024, 65535]
#   # Any port still not matched by a rule will not be forwarded (ignored)

# ===================================================================== #
# END OF TEMPLATE
# ===================================================================== #
```

```shell
# Make sure to delete existing Lima VM, if any
$ limactl delete default
INFO[0000] The QEMU process seems already stopped
INFO[0000] The host agent process seems already stopped
INFO[0000] Removing *.pid *.sock under "/Users/xxxx/.lima/default"
INFO[0000] Removing "/Users/xxxx/.lima/default/ga.sock"
INFO[0000] Deleted "default" ("/Users/xxxx/.lima/default")

$ limactl start default.yaml
? Creating an instance "default" Proceed with the default configuration
INFO[0002] Downloading "https://github.com/containerd/nerdctl/releases/download/v0.11.1/nerdctl-full-0.11.1-linux-arm64.tar.gz" (sha256:e2c8d0417b2fb79919f22a818813c646ad7ce0e600a951b6bac98340650e4435)
INFO[0002] Using cache "/Users/xxxx/Library/Caches/lima/download/by-url-sha256/cf356d64aa826ad177396feef432a14e0df5b2bb998191eff281541d329ff12c/data"
INFO[0002] Attempting to download the image from "~/Downloads/hirsute-server-cloudimg-arm64.img"
INFO[0002] Attempting to download the image from "https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-arm64.img"
INFO[0003] Using cache "/Users/xxxx/Library/Caches/lima/download/by-url-sha256/c40afd2acd5f2078759e01e119168213674615fff0701d53bc2bf69e5be3bf69/data"
INFO[0004] [hostagent] Starting QEMU (hint: to watch the boot progress, see "/Users/xxxx/.lima/default/serial.log")
INFO[0004] SSH Local Port: 60022
INFO[0004] [hostagent] Waiting for the essential requirement 1 of 4: "ssh"
INFO[0024] [hostagent] The essential requirement 1 of 4 is satisfied
INFO[0024] [hostagent] Waiting for the essential requirement 2 of 4: "sshfs binary to be installed"
INFO[0036] [hostagent] The essential requirement 2 of 4 is satisfied
INFO[0036] [hostagent] Waiting for the essential requirement 3 of 4: "/etc/fuse.conf to contain \"user_allow_other\""
INFO[0048] [hostagent] The essential requirement 3 of 4 is satisfied
INFO[0048] [hostagent] Waiting for the essential requirement 4 of 4: "the guest agent to be running"
INFO[0048] [hostagent] The essential requirement 4 of 4 is satisfied
INFO[0048] [hostagent] Mounting "/Users/xxxx"
INFO[0048] [hostagent] Mounting "/tmp/lima"
INFO[0048] [hostagent] Waiting for the optional requirement 1 of 2: "systemd must be available"
INFO[0048] [hostagent] Forwarding "/run/user/505/lima-guestagent.sock" (guest) to "/Users/xxxx/.lima/default/ga.sock" (host)
INFO[0048] [hostagent] Not forwarding TCP 127.0.0.53:53
INFO[0048] [hostagent] Not forwarding TCP 0.0.0.0:22
INFO[0048] [hostagent] The optional requirement 1 of 2 is satisfied
INFO[0048] [hostagent] Not forwarding TCP [::]:22
INFO[0048] [hostagent] Waiting for the optional requirement 2 of 2: "containerd binaries to be installed"
INFO[0051] [hostagent] The optional requirement 2 of 2 is satisfied
INFO[0051] READY. Run `lima` to open the shell.
```

Create a plain text HTML file for nginx.

```shell
$ pwd
/Users/xxxx/lima-test-directory

$ echo "replaced nginx content is here! yay!" > index.html 
```

Run nginx container with mount volume

```shell
lima nerdctl run -d -v $(pwd):/usr/share/nginx/html:ro -p 127.0.0.1:8080:80 nginx:alpine
```
​
