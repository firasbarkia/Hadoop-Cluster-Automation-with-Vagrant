# Hadoop Cluster on Vagrant

This project provides a Vagrantfile to automatically set up a three-node Hadoop cluster using VirtualBox. The cluster consists of one NameNode and two DataNodes configured for HDFS and YARN.

## Overview

The setup creates a fully functional Hadoop 3.3.6 cluster with:
- 1 NameNode (master)
- 2 DataNodes (workers)
- HDFS configured with replication factor of 2
- YARN for resource management
- MapReduce framework

## Prerequisites

- [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (6.1 or newer)
- [Vagrant](https://www.vagrantup.com/downloads) (2.2 or newer)
- At least 8GB of RAM available on the host machine
- At least 15GB of free disk space

## Network Configuration

The cluster uses a private network with the following IP addresses:
- NameNode: 192.168.50.10
- DataNode1: 192.168.50.11
- DataNode2: 192.168.50.12

All the necessary Hadoop ports are configured in the firewall rules to allow communication between nodes.

## Quick Start

1. Clone this repository:
   ```bash
   git clone https://github.com/yourusername/hadoop-cluster-vagrant.git
   cd hadoop-cluster-vagrant
   ```

2. Start the VMs:
   ```bash
   vagrant up
   ```
   This will create and provision all three nodes, which may take some time.

3. SSH into the NameNode:
   ```bash
   vagrant ssh namenode
   ```

4. Start the Hadoop services:
   ```bash
   ./start-hadoop.sh
   ```

5. Verify the cluster is running:
   ```bash
   hdfs dfsadmin -report
   yarn node -list
   ```

## Web UIs

Once the cluster is running, you can access the following web interfaces:
- HDFS NameNode: http://192.168.50.10:9870
- YARN ResourceManager: http://192.168.50.10:8088

## Stopping the Cluster

To stop the Hadoop services:
```bash
vagrant ssh namenode
/opt/hadoop/sbin/stop-yarn.sh
/opt/hadoop/sbin/stop-dfs.sh
```

To shut down the VMs:
```bash
vagrant halt
```

To completely remove the VMs:
```bash
vagrant destroy
```

## Configuration Details

### Resources

Each VM is configured with:
- 2 CPU cores
- 2GB of RAM

These settings can be adjusted in the Vagrantfile.

### Hadoop Configuration

- HDFS is configured with a replication factor of 2
- The NameNode and DataNodes data directories are located at `/opt/hadoop/data/`
- SSH keys are automatically configured for passwordless login between nodes
- The Hadoop configuration files are shared between nodes via the `/vagrant/hadoop-config/` directory

## Modifying the Cluster

### Adding More DataNodes

To add more DataNodes, edit the Vagrantfile and duplicate the DataNode configuration section, updating the hostname and IP address. You'll also need to add the new node to the `/etc/hosts` entries and the `workers` file.

### Changing Hadoop Configuration

To modify Hadoop settings, edit the XML configuration files generated in the NameNode provisioning script. Then run `vagrant provision` to apply the changes.

## Troubleshooting

### Connection Issues

If nodes cannot communicate, verify:
1. The firewall settings allow all necessary Hadoop ports
2. The `/etc/hosts` files on all nodes have correct entries
3. SSH passwordless authentication is working properly

### Services Not Starting

If Hadoop services don't start properly:
1. Check the logs in `/opt/hadoop/logs/`
2. Verify JAVA_HOME is correctly set
3. Ensure all necessary directory permissions are correct

## License

This project is licensed under the MIT License - see the LICENSE file for details.
