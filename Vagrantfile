Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.ssh.insert_key = false

  config.vm.provider "virtualbox" do |vb|
    # Increased resources for better performance
    vb.memory = "2048"
    vb.cpus = 2
  end

  # Clean up previous shared files on each provision
  config.trigger.before :up do |trigger|
    trigger.name = "Cleaning up previous shared files"
    trigger.run = {inline: "rm -f namenode.pub hadoop-config/*"}
    trigger.on_error = :continue
  end

  # NameNode configuration
  config.vm.define "namenode" do |namenode|
    namenode.vm.hostname = "namenode"
    namenode.vm.network "private_network", ip: "192.168.50.10"

    namenode.vm.provision "shell", inline: <<-SHELL
      # Update and install required packages
      sudo apt-get update
      sudo apt-get install -y openssh-server openjdk-11-jdk wget

      # Configure firewall for Hadoop services
      sudo apt-get install -y ufw
      sudo ufw --force reset
      sudo ufw default deny incoming
      sudo ufw default allow outgoing
      sudo ufw allow 22/tcp
      sudo ufw allow from 192.168.50.0/24 to any
      # HDFS ports
      sudo ufw allow 9000/tcp
      sudo ufw allow 9870/tcp
      sudo ufw allow 9864/tcp
      sudo ufw allow 50010/tcp
      sudo ufw allow 50020/tcp
      # YARN ports
      sudo ufw allow 8032/tcp
      sudo ufw allow 8088/tcp
      sudo ufw allow 8040:8042/tcp
      sudo ufw --force enable

      # Setup passwordless SSH to self
      mkdir -p /home/vagrant/.ssh
      chmod 700 /home/vagrant/.ssh
      rm -f /home/vagrant/.ssh/id_rsa /home/vagrant/.ssh/id_rsa.pub
      ssh-keygen -t rsa -b 4096 -f /home/vagrant/.ssh/id_rsa -N ""
      cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
      chmod 600 /home/vagrant/.ssh/id_rsa /home/vagrant/.ssh/id_rsa.pub /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh
      
      # Configure SSH for easier connections between nodes
      echo "Host namenode datanode1 datanode2 192.168.50.*
        StrictHostKeyChecking no
        UserKnownHostsFile=/dev/null
        IdentityFile /home/vagrant/.ssh/id_rsa" > /home/vagrant/.ssh/config
      chmod 600 /home/vagrant/.ssh/config
      chown vagrant:vagrant /home/vagrant/.ssh/config

      # Correct /etc/hosts entries
      sudo sed -i '/^192.168.50./d' /etc/hosts
      echo "192.168.50.10 namenode" | sudo tee -a /etc/hosts
      echo "192.168.50.11 datanode1" | sudo tee -a /etc/hosts
      echo "192.168.50.12 datanode2" | sudo tee -a /etc/hosts
      
      # Create shared directory
      mkdir -p /vagrant/hadoop-config
      cp /home/vagrant/.ssh/id_rsa.pub /vagrant/namenode.pub

      # Install Hadoop
      if [ ! -d "/opt/hadoop" ]; then
        wget -q https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
        sudo tar -xzf hadoop-3.3.6.tar.gz -C /opt/
        sudo mv /opt/hadoop-3.3.6 /opt/hadoop
        sudo chown -R vagrant:vagrant /opt/hadoop
      fi

      # Determine JAVA_HOME
      export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
      echo "Found JAVA_HOME: $JAVA_HOME"

      # Create Hadoop environment file
      sudo bash -c "cat > /etc/profile.d/hadoop.sh << EOF
#!/bin/bash
export JAVA_HOME=$JAVA_HOME
export HADOOP_HOME=/opt/hadoop
export PATH=\\\$PATH:\\\$JAVA_HOME/bin:\\\$HADOOP_HOME/bin:\\\$HADOOP_HOME/sbin
EOF"
      sudo chmod +x /etc/profile.d/hadoop.sh
      source /etc/profile.d/hadoop.sh

      # Set JAVA_HOME in Hadoop config
      echo "export JAVA_HOME=$JAVA_HOME" >> /opt/hadoop/etc/hadoop/hadoop-env.sh

      # Create Hadoop directories
      sudo rm -rf /opt/hadoop/data
      mkdir -p /opt/hadoop/data/namenode
      mkdir -p /opt/hadoop/data/datanode
      chown -R vagrant:vagrant /opt/hadoop

      # Generate Hadoop configs
      cat > /opt/hadoop/etc/hadoop/core-site.xml << EOF
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://namenode:9000</value>
  </property>
</configuration>
EOF

      cat > /opt/hadoop/etc/hadoop/hdfs-site.xml << EOF
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/opt/hadoop/data/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/opt/hadoop/data/datanode</value>
  </property>
  <property>
    <name>dfs.permissions</name>
    <value>false</value>
  </property>
</configuration>
EOF

      cat > /opt/hadoop/etc/hadoop/mapred-site.xml << EOF
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.application.classpath</name>
    <value>\$HADOOP_HOME/share/hadoop/mapreduce/*:\$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
  </property>
</configuration>
EOF

      cat > /opt/hadoop/etc/hadoop/yarn-site.xml << EOF
<?xml version="1.0"?>
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>namenode</value>
  </property>
</configuration>
EOF

      # Workers file (note: correct for Hadoop 3.x)
      echo "namenode" > /opt/hadoop/etc/hadoop/workers
      echo "datanode1" >> /opt/hadoop/etc/hadoop/workers
      echo "datanode2" >> /opt/hadoop/etc/hadoop/workers

      # Share configs and SSH key
      mkdir -p /vagrant/hadoop-config
      cp /opt/hadoop/etc/hadoop/*.xml /vagrant/hadoop-config/
      cp /opt/hadoop/etc/hadoop/workers /vagrant/hadoop-config/
      cp /home/vagrant/.ssh/id_rsa /vagrant/ssh_key
      chmod 600 /vagrant/ssh_key

      # Format HDFS with explicit environment
      sudo -u vagrant bash -c "source /etc/profile.d/hadoop.sh && echo 'Y' | /opt/hadoop/bin/hdfs namenode -format"
      
      # Create startup script
      cat > /home/vagrant/start-hadoop.sh << EOF
#!/bin/bash
source /etc/profile.d/hadoop.sh
echo "Starting Hadoop services..."
/opt/hadoop/sbin/start-dfs.sh
/opt/hadoop/sbin/start-yarn.sh
echo "Hadoop cluster status:"
/opt/hadoop/bin/hdfs dfsadmin -report
EOF
      chmod +x /home/vagrant/start-hadoop.sh
      chown vagrant:vagrant /home/vagrant/start-hadoop.sh
      
      echo "Hadoop cluster setup complete on namenode. After all VMs are provisioned, run './start-hadoop.sh' to start the Hadoop services."
    SHELL
  end

  # DataNode 1 configuration
  config.vm.define "datanode1" do |datanode|
    datanode.vm.hostname = "datanode1"
    datanode.vm.network "private_network", ip: "192.168.50.11"

    datanode.vm.provision "shell", inline: <<-SHELL
      # Update and install required packages
      sudo apt-get update
      sudo apt-get install -y openssh-server openjdk-11-jdk wget

      # Configure firewall for Hadoop services
      sudo apt-get install -y ufw
      sudo ufw --force reset
      sudo ufw default deny incoming
      sudo ufw default allow outgoing
      sudo ufw allow 22/tcp
      sudo ufw allow from 192.168.50.0/24 to any
      # HDFS ports
      sudo ufw allow 9000/tcp
      sudo ufw allow 9870/tcp
      sudo ufw allow 9864/tcp
      sudo ufw allow 50010/tcp
      sudo ufw allow 50020/tcp
      # YARN ports
      sudo ufw allow 8032/tcp
      sudo ufw allow 8088/tcp
      sudo ufw allow 8040:8042/tcp
      sudo ufw --force enable

      # Set up SSH access for namenode
      mkdir -p /home/vagrant/.ssh
      chmod 700 /home/vagrant/.ssh
      touch /home/vagrant/.ssh/authorized_keys
      
      # Wait for namenode.pub with timeout
      COUNTER=0
      MAX_RETRIES=30
      until [ -f /vagrant/namenode.pub ] || [ $COUNTER -eq $MAX_RETRIES ]; do
        echo "Waiting for namenode SSH key ($COUNTER/$MAX_RETRIES)..."
        sleep 5
        COUNTER=$((COUNTER+1))
      done
      
      if [ -f /vagrant/namenode.pub ]; then
        cat /vagrant/namenode.pub >> /home/vagrant/.ssh/authorized_keys
        chmod 600 /home/vagrant/.ssh/authorized_keys
        chown -R vagrant:vagrant /home/vagrant/.ssh
        echo "SSH key configured successfully."
      else
        echo "ERROR: Could not find namenode SSH key after $MAX_RETRIES attempts."
        exit 1
      fi

      # Correct /etc/hosts entries
      sudo sed -i '/^192.168.50./d' /etc/hosts
      echo "192.168.50.10 namenode" | sudo tee -a /etc/hosts
      echo "192.168.50.11 datanode1" | sudo tee -a /etc/hosts
      echo "192.168.50.12 datanode2" | sudo tee -a /etc/hosts

      # Install Hadoop
      if [ ! -d "/opt/hadoop" ]; then
        wget -q https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
        sudo tar -xzf hadoop-3.3.6.tar.gz -C /opt/
        sudo mv /opt/hadoop-3.3.6 /opt/hadoop
        sudo chown -R vagrant:vagrant /opt/hadoop
      fi

      # Determine JAVA_HOME
      export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
      echo "Found JAVA_HOME: $JAVA_HOME"

      # Create Hadoop environment file
      sudo bash -c "cat > /etc/profile.d/hadoop.sh << EOF
#!/bin/bash
export JAVA_HOME=$JAVA_HOME
export HADOOP_HOME=/opt/hadoop
export PATH=\\\$PATH:\\\$JAVA_HOME/bin:\\\$HADOOP_HOME/bin:\\\$HADOOP_HOME/sbin
EOF"
      sudo chmod +x /etc/profile.d/hadoop.sh
      source /etc/profile.d/hadoop.sh

      # Set JAVA_HOME in Hadoop config
      echo "export JAVA_HOME=$JAVA_HOME" >> /opt/hadoop/etc/hadoop/hadoop-env.sh

      # Wait for Hadoop configs with timeout
      COUNTER=0
      MAX_RETRIES=30
      until [ -f /vagrant/hadoop-config/core-site.xml ] || [ $COUNTER -eq $MAX_RETRIES ]; do
        echo "Waiting for Hadoop configuration files ($COUNTER/$MAX_RETRIES)..."
        sleep 5
        COUNTER=$((COUNTER+1))
      done
      
      if [ -f /vagrant/hadoop-config/core-site.xml ]; then
        mkdir -p /opt/hadoop/etc/hadoop
        cp /vagrant/hadoop-config/* /opt/hadoop/etc/hadoop/
        echo "Hadoop configs copied successfully."
      else
        echo "ERROR: Could not find Hadoop configurations after $MAX_RETRIES attempts."
        exit 1
      fi

      # Create data directory
      sudo rm -rf /opt/hadoop/data
      mkdir -p /opt/hadoop/data/datanode
      chown -R vagrant:vagrant /opt/hadoop
      
      echo "DataNode1 setup complete."
    SHELL
  end

  # DataNode 2 configuration
  config.vm.define "datanode2" do |datanode|
    datanode.vm.hostname = "datanode2"
    datanode.vm.network "private_network", ip: "192.168.50.12"

    datanode.vm.provision "shell", inline: <<-SHELL
      # Update and install required packages
      sudo apt-get update
      sudo apt-get install -y openssh-server openjdk-11-jdk wget

      # Configure firewall for Hadoop services
      sudo apt-get install -y ufw
      sudo ufw --force reset
      sudo ufw default deny incoming
      sudo ufw default allow outgoing
      sudo ufw allow 22/tcp
      sudo ufw allow from 192.168.50.0/24 to any
      # HDFS ports
      sudo ufw allow 9000/tcp
      sudo ufw allow 9870/tcp
      sudo ufw allow 9864/tcp
      sudo ufw allow 50010/tcp
      sudo ufw allow 50020/tcp
      # YARN ports
      sudo ufw allow 8032/tcp
      sudo ufw allow 8088/tcp
      sudo ufw allow 8040:8042/tcp
      sudo ufw --force enable

      # Set up SSH access for namenode
      mkdir -p /home/vagrant/.ssh
      chmod 700 /home/vagrant/.ssh
      touch /home/vagrant/.ssh/authorized_keys
      
      # Wait for namenode.pub with timeout
      COUNTER=0
      MAX_RETRIES=30
      until [ -f /vagrant/namenode.pub ] || [ $COUNTER -eq $MAX_RETRIES ]; do
        echo "Waiting for namenode SSH key ($COUNTER/$MAX_RETRIES)..."
        sleep 5
        COUNTER=$((COUNTER+1))
      done
      
      if [ -f /vagrant/namenode.pub ]; then
        cat /vagrant/namenode.pub >> /home/vagrant/.ssh/authorized_keys
        chmod 600 /home/vagrant/.ssh/authorized_keys
        chown -R vagrant:vagrant /home/vagrant/.ssh
        echo "SSH key configured successfully."
      else
        echo "ERROR: Could not find namenode SSH key after $MAX_RETRIES attempts."
        exit 1
      fi

      # Correct /etc/hosts entries
      sudo sed -i '/^192.168.50./d' /etc/hosts
      echo "192.168.50.10 namenode" | sudo tee -a /etc/hosts
      echo "192.168.50.11 datanode1" | sudo tee -a /etc/hosts
      echo "192.168.50.12 datanode2" | sudo tee -a /etc/hosts

      # Install Hadoop
      if [ ! -d "/opt/hadoop" ]; then
        wget -q https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
        sudo tar -xzf hadoop-3.3.6.tar.gz -C /opt/
        sudo mv /opt/hadoop-3.3.6 /opt/hadoop
        sudo chown -R vagrant:vagrant /opt/hadoop
      fi

      # Determine JAVA_HOME
      export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
      echo "Found JAVA_HOME: $JAVA_HOME"

      # Create Hadoop environment file
      sudo bash -c "cat > /etc/profile.d/hadoop.sh << EOF
#!/bin/bash
export JAVA_HOME=$JAVA_HOME
export HADOOP_HOME=/opt/hadoop
export PATH=\\\$PATH:\\\$JAVA_HOME/bin:\\\$HADOOP_HOME/bin:\\\$HADOOP_HOME/sbin
EOF"
      sudo chmod +x /etc/profile.d/hadoop.sh
      source /etc/profile.d/hadoop.sh

      # Set JAVA_HOME in Hadoop config
      echo "export JAVA_HOME=$JAVA_HOME" >> /opt/hadoop/etc/hadoop/hadoop-env.sh

      # Wait for Hadoop configs with timeout
      COUNTER=0
      MAX_RETRIES=30
      until [ -f /vagrant/hadoop-config/core-site.xml ] || [ $COUNTER -eq $MAX_RETRIES ]; do
        echo "Waiting for Hadoop configuration files ($COUNTER/$MAX_RETRIES)..."
        sleep 5
        COUNTER=$((COUNTER+1))
      done
      
      if [ -f /vagrant/hadoop-config/core-site.xml ]; then
        mkdir -p /opt/hadoop/etc/hadoop
        cp /vagrant/hadoop-config/* /opt/hadoop/etc/hadoop/
        echo "Hadoop configs copied successfully."
      else
        echo "ERROR: Could not find Hadoop configurations after $MAX_RETRIES attempts."
        exit 1
      fi

      # Create data directory
      sudo rm -rf /opt/hadoop/data
      mkdir -p /opt/hadoop/data/datanode
      chown -R vagrant:vagrant /opt/hadoop
      
      echo "DataNode2 setup complete."
    SHELL
  end
end