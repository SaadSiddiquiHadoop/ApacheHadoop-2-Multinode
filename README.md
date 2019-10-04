# ApacheHadoop-2-Multinode

# Connect To the DataCenter with Public Key or Private in case of windows through Putty.

ssh -i Downloads/file.pem ubuntu@public_dns_address

# Update the system
sudo apt-get update && sudo apt-get dist-upgrade -y

# Copy public key on to the DataCenter main server
scp -i multi.pem multi.pem ubuntu@public_dns:~/.ssh

# Create a Hadoop user for accessing HDFS
sudo addgroup hadoop
sudo adduser hduser --ingroup hadoop 
sudo adduser hduser sudo
sudo su hduser

# Create local key
ssh-keygen
cat id_rsa.pub >> authorized_keys

# Copy the instance public key (multi.pem) to hduser's directory
sudo su
cp /home/ubuntu/.ssh/multi.pem /home/hduser/.ssh/
chown hduser:hadoop /home/hduser/.ssh/multi.pem
exit

# Install Java 8
sudo apt-get install -y python-software-properties debconf-utils
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get install -y oracle-java8-installer

# Download and Install Hadoop
wget -c http://www-us.apache.org/dist/hadoop/common/stable/hadoop-2.9.0.tar.gz
tar -xzvf hadoop-2.7.4.tar.gz
sudo mv hadoop-2.7.4 /usr/local/hadoop
sudo chown -R hduser:hadoop /usr/local/hadoop

# Set Enviornment Variable
readlink -f $(which java)
nano ~/.bashrc
# -- HADOOP ENVIRONMENT VARIABLES START -- #
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
export PATH=$PATH:/usr/local/hadoop/bin/
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export PDSH_RCMD_TYPE=ssh
# -- HADOOP ENVIRONMENT VARIABLES END -- #

$ source ~/.bashrc

cd /usr/local/hadoop/etc/hadoop/

#Update hadoop-env.sh
nano hadoop-env.sh
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export HADOOP_LOG_DIR=/var/log/hadoop
sudo mkdir /var/log/hadoop/
sudo chown -R hduser:hadoop /var/log/hadoop

#Disable IPV6
sudo nano /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
sudo sysctl -p

#Disable FireWall iptables
sudo iptables -L -n
sudo ufw status
sudo ufw disable

#Disabling Transparent Hugepage Compaction
--->> #Red Hat/CentOS: /sys/kernel/mm/redhat_transparent_hugepage/defrag
--->> #Ubuntu/Debian, OEL, SLES: /sys/kernel/mm/transparent_hugepage/defrag

cat /sys/kernel/mm/transparent_hugepage/defrag
sudo sed -i '/exit 0/d' /etc/rc.local

sudo su -c 'cat >>/etc/rc.local <<EOL
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
   echo never > /sys/kernel/mm/transparent_hugepage/defrag 
fi
exit 0
EOL'

sudo -i
source /etc/rc.local


# Set Swappiness
sudo sysctl -a | grep vm.swappiness
sudo sysctl vm.swappiness=0

# Configure NTP 
timedatectl status
timedatectl list-timezones
sudo timedatectl set-timezone Asia/Kolkata
sudo ntpq -p
sudo apt install ntp -y
sudo nano /etc/ntp.conf

##Configure SSH Password less logins
sudo su -c touch /home/hduser/.ssh/config; echo "Host *\n StrictHostKeyChecking no\n
UserKnownHostsFile=/dev/null" > /home/hduser/.ssh/config
sudo service ssh restart

# Configure .profile (make sure you are on NN) 
 nano .profile
 eval `ssh-agent` ssh-add /home/hduser/.ssh/abc.pem

 source .profile


----------****** Create a snapshot at this point ******-----------------
Create 4 nodes from this image

#sudo nano /etc/hosts and include these lines:FQDN
172.31.29.56 ip-172-31-29-56.ec2.internal nn
172.31.27.96 ip-172-31-27-96.ec2.internal rm
172.31.18.137 ip-172-31-18-137.ec2.internal 1dn
172.31.23.79 ip-172-31-23-79.ec2.internal 2dn
172.31.16.236 ip-172-31-16-236.ec2.internal 3dn

Do this for all nodes

# Install and Configure dsh
sudo apt install dsh -y
sudo nano /etc/dsh/machines.list
#localhost
nn
rm
1dn
2dn
3dn
dsh -a uptime
dsh -a source .profile

cd /usr/local/hadoop/etc/hadoop

# Configure masters and slaves
nano masters
rm

nano slaves
1dn
2dn
3dn

#Update core-site.xml
nano core-site.xml
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://nn:9000</value>
  </property>

#Update hdfs-site.xml on name node 
mkdir -p /usr/local/hadoop/data/hdfs/namenode
nano hdfs-site.xml
<property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///usr/local/hadoop/data/hdfs/namenode</value>
  </property>
   <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///usr/local/hadoop/data/hdfs/datanode</value>
  </property> 


#Create proper directories on datanode's
dsh -m 1dn,2dn,3dn mkdir -p /usr/local/hadoop/data/hdfs/datanode
 


#Update yarn-site.xml
nano yarn-site.xml
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>rm</value>
  </property>

#Update mapred-site.xml

cp mapred-site.xml.template mapred-site.xml
nano mapred-site.xml
<property>
    <name>mapreduce.jobtracker.address</name>
    <value>rm:54311</value>
  </property>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>

sudo chown -R hduser:hadoop $HADOOP_HOME

#SCP all the files
cd /usr/local/hadoop/etc/hadoop
For all nodes
scp core-site.xml hdfs-site.xml mapred-site.xml yarn-site.xml slaves rm:/usr/local/hadoop/etc/hadoop



#Format Namenode

hdfs namenode -format


# Start the cluster
start-dfs.sh
start-yarn.sh
 
dsh -a jps


hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/ubuntu
hdfs dfs -put hadoop-2.8.2.tar.gz /user/ubuntu
hdfs dfs -ls 
hdfs dfs -ls -R

yarn jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.2.jar pi  5 10


