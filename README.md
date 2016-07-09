# OpenStack Virtual cluster with Ansible.
Setting up a bastion node instance with elasticluster and SLURM queue. This is quite an involved process and something you cannot follow blindly. Please at least read how OpenStack works and what elasticluster/slurm does. Familarise yourself with public/private key generation and what a virtual machine is.

This guide is to configure your own openstack project from scratch. If you are simply using the elasticluster I have set up, and do not have a project of your own - *DO NOT* run any of the commands below, apart from logging in and submitting jobs. Write down the VPN and bastion node instance IP and perhaps refer to the SLURM commands to submit jobs. 

The current setup is to access emedlab using a VPN connection:

```https://vpn.emedlab.ac.uk/ ```

Once connected to the VPN through the Cisco AnyConnect client we are able to access both the openStack emedlab web interface/dashboard:

Our group is rna-seq-map-ref - Browse to the OpenStack Dashboard using the following link:

``` http://10.0.42.197/ ```

The bastion instance (using your own ssh private key):

```ssh -i id_rsa username@10.2.223.19```

The bastion instance is running CentOS and has root level access. If you have your own bastion instance running CentOS, it will be necessary to install everything you might be familar with. 

The bastion node is also important as it is the instance that has access to the OpenStack public API. This is used to instantiate VM images, configure them and to check what is running, which can also be done via the web dashboard. However, to run things such as elasticluster with a SLURM queue, command line level access is required.

To interact with the OpenStack API it is neccessary to touch a configuration file - supplied above:

``` rna-seq-map-ref-openrc.sh ```

This configuration file needs to be modified with your username and password. I have added this to my bashrc profile so it runs automatically when I connnect.

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

Make sure you have ansible version > 2.x. Else it will lead to an error as it will try and use a class called 'Package' and fail (because it doesn't exist).

At this point you can launch a cluster with as many instances as you like, and can SSH into any of them easily.

I have setup a cluster of 18 slaves and 1 headnode. From the headnode you can access the queue. You can either use SLURM or the open source version of SGE (qstat/qrsh/qsub etc).

I added a 29Tb volume to the headnode instance of an example elasticluster setup so that it's possible to have an network file system (NFS) that all slave instances can write to.

This required manually creating a filesystem, and due to it being 29Tb (v.large) GPT was used:

```
sudo su
parted /dev/vdb mklabel gpt
parted /dev/vdb mkpart primary xfs 1 -1
mkfs.xfs -L ee /dev/vdb1
mount /dev/vdb1/ /mnt
cd ~/
ln -s /mnt/shared_data/ scratch
```
A mounted 29Tb volume is now available to write to. This can hold all the data we want to work on / share. I have called it scratch.

This should be mounted on the headnode now. To turn this into a network file system (NFS) for all other compute nodes to be able to access and write to it, add to /etc/exports

```/mnt/shared_data compute001(rw,no_root_squash,async,no_subtree_check,crossmnt) compute002(rw,no_root_squash,async,no_subtree_check,crossmnt) compute003(rw,no_root_squash,async,no_subtree_check,crossmnt) compute004(rw,no_root_squash,async,no_subtree_check,crossmnt) compute005(rw,no_root_squash,async,no_subtree_check,crossmnt) compute006(rw,no_root_squash,async,no_subtree_check,crossmnt) compute007(rw,no_root_squash,async,no_subtree_check,crossmnt) compute008(rw,no_root_squash,async,no_subtree_check,crossmnt) compute009(rw,no_root_squash,async,no_subtree_check,crossmnt) compute010(rw,no_root_squash,async,no_subtree_check,crossmnt) compute011(rw,no_root_squash,async,no_subtree_check,crossmnt) compute012(rw,no_root_squash,async,no_subtree_check,crossmnt) compute013(rw,no_root_squash,async,no_subtree_check,crossmnt) compute014(rw,no_root_squash,async,no_subtree_check,crossmnt) compute015(rw,no_root_squash,async,no_subtree_check,crossmnt) compute016(rw,no_root_squash,async,no_subtree_check,crossmnt) compute017(rw,no_root_squash,async,no_subtree_check,crossmnt) compute018(rw,no_root_squash,async,no_subtree_check,crossmnt)```

Invoke ansible to mount the drive on all compute nodes:

```ansible -i ~/.elasticluster/storage/ansible-inventory.slurm+pvfs2 slurm_hosts -m mount -s -a “name=media/ src=frontend001:/media fstype=nfs state=mounted""```

We now have a drive that is accessible to all nodes '/mnt/shared_data/'.

Software can be installed to all compute nodes by doing the following (example R)

``` ansible -i ~/.elasticluster/storage/ansible-inventory.slurm $slurm_hosts -s -m yum -a "name='R' state=present"```

Where ```$slurm_hosts``` is a variable consisting of all the node names (frontend001,compute001,compute002...computeN)

I have created an ansible playbook that will install R-packages automatically on all hosts by running:

```ansible-playbook r_packages.yml -i .elasticluster/storage/ansible-inventory.slurm ```

You should now have R, java, python and basic/neccessary R packages such as lme4, Matrix, and data.table.

Some tools, such as adapter trimming software use perl - amongst many other bioinformatic tools. To install perl, cpan and perl modules, I have added an ansible playbook to install across the cluster:

```ansible-playbook perl.yml -i .elasticluster/storage/ansible-inventory.slurm ```

##SLURM commands and batch jobs

Use the following header for your batch job submissions:

```
#!/bin/bash 
# 
#SBATCH -p general 
# partition (queue) 
#SBATCH -N 1 
# number of nodes 
#SBATCH -n 1 
# number of cores 
#SBATCH --mem 100 
# memory pool for all cores 
#SBATCH -t 0-2:00 
# time (D-HH:MM) 
#SBATCH -o slurm.%N.%j.out 
# STDOUT #SBATCH -e slurm.%N.%j.err 
# STDERR #SBATCH --mail-type=END,FAIL # notifications for job done & fail 
#SBATCH --mail-user=myemail@test.kcl.ac.uk # send-to address 
```
Useful commands to use:

``` sbatch myscript.sh ``` 

``` squeue -u <username> ```

``` scancel <jobid> ```
