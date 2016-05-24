# OpenStack
Setting up a bastion node instance with elasticluster and SLURM queue.

The current setup is to access emedlab using a VPN connection:

```http://vpn.emedlab.ac.uk/ ```

Once connected to the VPN through the Cisco AnyConnect client we are able to access both the openStack emedlab web interface/dashboard:

Our group is rna-seq-map-ref
``` http://10.0.42.197/ ```

and the bastion instance:

```'ssh username@10.2.223.19'```

The bastion instance is running CentOS and has root level access. If you have your own bastion instance running CentOS, it will be necessary to install everything you might be fimilar with. CentOS uses yum rather than ```‘apt-get’```, and programs such as nano, wget etc are not installed by default. Install them using: ```‘sudo yum install nano’```.
This bastion node is also important as it is the instance that has access to the OpenStack public API. This is used to instantiate VM images, configure them, check what is running etc which can also be done via the web dashboard. However, to run things such as elasticluster and SLURM, command line level access is required.
