# OpenStack
Setting up a bastion node instance with elasticluster and SLURM queue.

The current setup is to access emedlab using a VPN connection:

```http://vpn.emedlab.ac.uk/ ```

Once connected to the VPN through the Cisco AnyConnect client we are able to access both the openStack emedlab web interface/dashboard:

Our group is rna-seq-map-ref
``` http://10.0.42.197/ ```

and the bastion instance:

```ssh username@10.2.223.19```

The bastion instance is running CentOS and has root level access. If you have your own bastion instance running CentOS, it will be necessary to install everything you might be fimilar with. CentOS uses yum rather than ```apt-get```, and programs such as nano, wget etc are not installed by default. Install them using: ```sudo yum install nano```.
This bastion node is also important as it is the instance that has access to the OpenStack public API. This is used to instantiate VM images, configure them, check what is running etc which can also be done via the web dashboard. However, to run things such as elasticluster and SLURM, command line level access is required.

To interact with the OpenStack API it is neccessary to touch a configuration file - supplied above:

``` rna-seq-map-ref-openrc.sh ```
This configuration file needs to be modified with your username and password

## Elasticluster & SLURM

install elasticluster via pip and any dependencies it has. This is only neccessary if you're using a bastion instance that doesn't have this already configured (for example if you're using my bastion instance).

Elasticluster allows one to start several instances at the same time, all with settings configured to the users need (I.e. I want 8 instances of ubuntu, each with 32GB of RAM and 8 cores each). It also sets up a headnode. The headnode can be used to run a queue system, much like a traditional HPC environment.

Information on elasticluster can be found here:

```http://elasticluster.readthedocs.io/en/latest/```

