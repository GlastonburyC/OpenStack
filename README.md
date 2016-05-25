# OpenStack
Setting up a bastion node instance with elasticluster and SLURM queue. This is quite an involved process and something you cannot follow blindly. Please at least read how OpenStack works and what elasticluster/slurm does. Familarise yourself with public/private key generation and what a virtual machine is.

The current setup is to access emedlab using a VPN connection:

```http://vpn.emedlab.ac.uk/ ```

Once connected to the VPN through the Cisco AnyConnect client we are able to access both the openStack emedlab web interface/dashboard:

Our group is rna-seq-map-ref

``` http://10.0.42.197/ ```

and the bastion instance (using your own ssh private key):

```ssh -i id_rsa username@10.2.223.19```

The bastion instance is running CentOS and has root level access. If you have your own bastion instance running CentOS, it will be necessary to install everything you might be fimilar with. CentOS uses yum rather than ```apt-get```, and programs such as nano, wget etc are not installed by default. Install them using: ```sudo yum install nano```.

I added a 25Tb volume to the bastion instance so that it's possible to eventually have an network file system (NFS) that all slave instances can write to.

This required manually creating a filesystem, and due to it being 25Tb (v.large) GPT was used:

```
sudo su
parted /dev/vdb mklabel gpt
parted /dev/vdb mkpart primary xfs 1 -1
mkfs.xfs -L ee /dev/vdb1
mount /dev/vdb1/ /mnt
cd ~/
ln -s /mnt/shared_data/ scratch
```
A mounted 25Tb volume is now available to write to on the bastion node. This can hold all the data we want to work on / share. I have called it scratch.

This bastion node is also important as it is the instance that has access to the OpenStack public API. This is used to instantiate VM images, configure them, check what is running etc which can also be done via the web dashboard. However, to run things such as elasticluster and SLURM, command line level access is required.


To interact with the OpenStack API it is neccessary to touch a configuration file - supplied above:

``` rna-seq-map-ref-openrc.sh ```

This configuration file needs to be modified with your username and password.

Once this is done, you can access useful information - For example, what flavour nodes are available/pre configured to use:

```nova flavor-list```

List images available:

```openstack image list```

For more commands see:

``` openstack --help ``` 

and also:

``` http://developer.openstack.org/api-guide/quick-start/ ```

## Elasticluster, MUNGE & SLURM

install elasticluster via github and any dependencies it has. The development version is required so that it can readily handle invalid SSL certificates that are present on emedlab (I found this out the hard way after installing the non-dev version). This process is only neccessary if you're using a bastion instance that doesn't have this already configured (for example if you're using my bastion instance all this is already done and you just need to learn how to use elasticluster and the SLURM queue). 

Elasticluster allows one to start several instances at the same time, all with settings configured to the users need (I.e. I want 8 instances of ubuntu, each with 32GB of RAM and 8 cores each). It also sets up a headnode. The headnode can be used to run a queue system, much like a traditional HPC environment.

Information on elasticluster can be found here:

``` http://elasticluster.readthedocs.io/en/latest/ ```

A have included a basic elasticluster config file which can be modified and which needs modifying depending on your requirements (RAM per instance, CPU per instance, how many instances, run time etc) This can be found in the following file and should be written as a file to home/username/.elasticluster/config:

``` config ```

The configuration file should be modified with your username and password, and ssh-key name. The ssh-key name needs to be added via the OpenStack dashboard via the 'Access and Security panel > key pairs' tab. Here you can copy your ssh public key.

MUNGE needs to be installed before SLURM, to act as a daemon on each instance so that you can readily ssh into each VM and access to each VM is automatically coordinated using MUNGE. Both MUNGE and SLURM can be installed using the following guide:

``` http://www.slothparadise.com/how-to-install-slurm-on-centos-7-cluster/ ```

I have already generated a SLURM configuration file so you don't have to:

``` slurm.conf ```

This configuration file should tell SLURM how many 'slave' instances you have, and which node is your head instance is - as generated by your elasticluster configuration file.

Once you have everything installed, it should be possible to start a virtual cluster:

``` elasticluster start slurm -v ```

By using the slothparadise guide, it's important you follow their suggestions and copy your MUNGE key onto each node - This can be done using an autoconfiguration script.

