# Vagrant-Ansible Sandbox
VA Sandbox (VAS) is an environment for development experimentation. It automates the creation of disposable VM environments that can be used to test ideas, configurations and settings, that otherwise would be painful to create and maintain manually.

Table of Contents
=================

  * [Vagrant\-Ansible Sandbox](#vagrant-ansible-sandbox)
    * [Prerequisites](#prerequisites)
    * [How is this Sandbox organized](#how-is-this-sandbox-organized)
    * [How to use it](#how-to-use-it)
        * [Step 1: Clone the Git Repo](#step-1-clone-the-git-repo)
        * [Step 2: Edit Vagrantfile](#step-2-edit-vagrantfile)
        * [Step 3: Start the VMs](#step-3-start-the-vms)
        * [Step 4: SSH into the mgmt node](#step-4-ssh-into-the-mgmt-node)
        * [Step 5: Use Ansible](#step-5-use-ansible)

## Prerequisites
This sandbox assumes the following software is already setup on your local laptop / workstation:

  + Virtualbox for creating and running VMs locally. Though one can use VMWare or KVM.
  + Vagrant
  + Access to Virtualbox images.


## How is this Sandbox organized
The sandbox provides a management node 'mgmt' VM and bunch of other VMs that can be wired together in flexible configurations. The management node is a ramp node and should be used to initiate all types of experimentation and is the only node that needs Ansible to be installed on it. This way, there is no need to install anything additional on the local laptop, and it makes a clean way to setup and tear down the sandboxes. Once the experimentations are done, all the VMs can be safely destroyed.

The sample Vagrantfile included here lists the management node:

```text
# create mgmt/on-ramp node
config.vm.define :mgmt do |mgmt_config|
    mgmt_config.vm.box = "centos-7.1-x64"
    mgmt_config.vm.hostname = "mgmt"
    mgmt_config.vm.network :private_network, ip: "192.168.56.120"
    mgmt_config.vm.provider "virtualbox" do |vb|
      vb.memory = "256"
    end
    mgmt_config.vm.provision :shell, path: "bootstrap-mgmt.sh"
end
```
Management node is provisioned via `bootstrap-mgmt.sh` shell script, after it is properly initialized by the underlying hypervisor. It:

  + Installs Ansible and some other useful tools on the management node,
  + Copies all the ansible playbooks in the `/home/vagrant/setup` directory.
  + Installs a kick-ass prompt. Frankly I got bored with the plain prompt which is not very useful, and
  + Updates the `/etc/hosts` file to list all the other nodes that the management node can reach.

There is a load-balancer VM configuration also provided as an example. If there is no need to a load balancer, then comment out the section from the Vagrantfile.

The Vagrantfile also lists the VM configurations of worker nodes as follows:

```text
# create some web servers
(1..2).each do |i|
  config.vm.define "node#{i}" do |node|
      node.vm.box = "centos-7.1-x64"
      node.vm.hostname = "node#{i}"
      node.vm.network :private_network, ip: "192.168.56.13#{i}"
      node.vm.network "forwarded_port", guest: 80, host: "808#{i}"
      node.vm.provider "virtualbox" do |vb|
        vb.memory = "256"
      end
  end
end
```
One can create as many VMs as needed by using the above construct.

## How to use it

### Step 1: Clone the Git Repo
After making sure that all prerequisites are met, clone the git repo:
```
$ git clone https://github.com/zeebluejay/vagrant-ansible-sandbox.git
```

Most common use case for this sandbox is to be able to create local VMs using Vagrant.

### Step 2: Edit Vagrantfile
```
$ vim Vagrantfile
```

Edit Vagrantfile to suit your needs for the number of VMs you need.

1. Make sure to edit the static network interfaces to match your own network subnets, if different.
2. Keep or remove the load-balancer VM.
3. Update the number of worker nodes needed for the setup. Make sure to update or remove the port forwarding of the worker nodes.
4. All the VMs are configured with 256MB RAM, which is enough for small scale experimentation. You can adjust the memory size if necessary.

### Step 3: Start the VMs
```
$ vagrant up
```
Wait for all the VMs to come up.

### Step 4: SSH into the mgmt node
```
$ vagrant ssh mgmt
```

### Step 5: Use Ansible

```
# Check ansible version
$ ansible --version

# Go to ansible playbooks directory
$ cd setup_trusted_nodes

# Edit the inventory file to reflect the proper number of nodes ansible has to provision. Comment the extra ones.
$ vim inventory.txt

# Run ansible playbook
$ ansible-playbook site.yml

```

> NOTE that **ansible.cfg** file has a special flag called `ask_pass=True`. This is only needed for the first run. It asks for **vagrant** username (password vagrant). It makes sense to comment out this line after the successful first run.

> NOTE you can optionally disable host checking in ansible and bypass the creation / update of known_hosts file altogether. In that case add `host_key_checking=True` to **ansible.cfg** file and comment out the role **establish_known_hosts** from **site.yml**.

####This is it! Happy experimentation!
