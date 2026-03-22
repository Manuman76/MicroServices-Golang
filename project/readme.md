# Multi-Node Docker Swarm Lab on Fedora

This guide provides a reproducible workflow for setting up a 3-node Docker Swarm cluster using **Vagrant** and **libvirt (KVM)** on Fedora Workstation.

## 1. Prerequisites

Install the native Fedora virtualization stack and Vagrant plugins:

```bash
# Install virtualization tools
sudo dnf install -y vagrant-libvirt libvirt qemu-kvm virt-manager

# Enable modular libvirt daemons (Required for Fedora 43+)
sudo systemctl enable --now virtqemud.socket virtqemud virtnetworkd virtstoraged

# Add your user to the libvirt group
sudo usermod -aG libvirt $USER
# IMPORTANT: Log out and back in for group changes to take effect.

```

### Check the Network Status

Run this to see if the network exists but is just "inactive":
Bash

```bash
sudo virsh net-list --all
```
- If it's "Inactive": Just start it and set it to autostart:
```Bash
sudo virsh net-start default
sudo virsh net-autostart default
```
- If it's Missing Entirely: You'll need to re-define it (see below).

### How to Re-create the Missing "default" Network

If net-list showed nothing, run these commands to restore the standard configuration:

Create a temporary XML file:
``` Bash
cat <<EOF > default_net.xml
<network>
    <name>default</name>
    <forward mode='nat'/>
    <bridge name='virbr0' stp='on' delay='0'/>
    <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
        <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
    </ip>
</network>
EOF
```
Define and start it:

```Bash
sudo virsh net-define default_net.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

## 2. Infrastructure Setup

Create a file named `Vagrantfile` in your project directory with the following configuration:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :libvirt do |libvirt|
    libvirt.driver = "qemu"
    libvirt.uri = "qemu:///system"
  end

  config.vm.box = "fedora/41-cloud-base"

  $script = <<-SCRIPT
    sudo dnf install -y moby-engine
    sudo systemctl enable --now docker
    sudo usermod -aG docker vagrant
  SCRIPT

  # Manager
  config.vm.define "manager" do |mgr|
    mgr.vm.hostname = "swarm-manager"
    # Added explicit IP to avoid the to_range nil error
    mgr.vm.network "private_network", ip: "192.168.121.10", libvirt__network_name: "default"
    mgr.vm.provision "shell", inline: $script
    mgr.vm.provider :libvirt do |lv|
      lv.memory = 4096
      lv.cpus = 2
    end
  end

  # Workers
  (1..2).each do |i|
    config.vm.define "worker#{i}" do |worker|
      worker.vm.hostname = "swarm-worker-#{i}"
      # Added explicit IP
      worker.vm.network "private_network", ip: "192.168.121.1#{i}", libvirt__network_name: "default"
      worker.vm.provision "shell", inline: $script
      worker.vm.provider :libvirt do |lv|
        lv.memory = 4096
        lv.cpus = 2
      end
    end
  end
```

## 3. Launching the Lab

```bash
# Initialize the VMs
vagrant up --provider=libvirt

# Check status
vagrant status

```

## 4. Initializing the Cluster

1. **Enter the Manager:**
```bash
vagrant ssh manager

```


2. **Identify the Private IP:**
Find the IP address on interface `ens6` (the `192.168.122.x` range):
```bash
ip addr show ens6

```


3. **Init Swarm:**
```bash
sudo docker swarm init --advertise-addr <ENS6_IP>

```


*Copy the `docker swarm join --token ...` command provided in the output.*

## 5. Joining Workers

Exit the manager and join each worker node to the cluster:

```bash
# Repeat for worker1 and worker2
vagrant ssh worker1
sudo <PASTE_JOIN_COMMAND_HERE>
exit

```

## 6. Verification

Back on the manager, verify all nodes are **Ready**:

```bash
vagrant ssh manager
sudo docker node ls

```

## 7. Deploy Stack

```bash
docker stack deploy -c swarm.yml myapp

```

## 8. Cleanup

When finished, destroy the VMs to free up system resources:

```bash
vagrant destroy -f

```

## Pro-Tip: Visualizing the Cluster

Deploy Visualizer as a Docker service. On the manager:

```Bash
sudo docker service create \
  --name=viz \
  --publish=8000:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer
```
Then, 192.168.122.111:8000 in a browser.
