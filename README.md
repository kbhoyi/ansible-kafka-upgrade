# Kafka Nodes Rolling Upgrade Using Ansible

This work is inspired from https://github.com/sanjeevmaheve/ansible-kafka-cluster . All Kafka nodes in a cluster can be upgraded from one minor version  to another version one at a time, keeping the traffic ongoing through the cluster. 

## How to use

In this version, its assumed that you have provisioned VMs with Ubuntu OS either on cloud or you local infrastructure before you choose to run ansible playbook.

### Pre-requisite

Install Ansible on the development machine/VM. There are number of ways, I followed using Python package.

```
$ sudo pip install ansible
$ git clone https://github.com/kbhoyi/ansible-kafka-upgrade
```

Below ansible playbook can help get the base setup done quickly

```
https://github.com/sanjeevmaheve/ansible-kafka-cluster
```

/etc/hosts file on all the kafka nodes should be updated with the peer node information.Replace <<username-virtual-machine>> with the username of the machine

```
sfuser@sfuser-virtual-machine:~/ansible-kafka-upgrade$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       <<username-virtual-machine>>

# The following lines are desirable for IPv6 capable hosts
192.168.0.159 <<username-virtual-machine>> kafka-2
192.168.0.160 <<username-virtual-machine>> kafka-1
192.168.0.163 <<username-virtual-machine>> kafka-consumer

```

Verify the consumer is recieving all messages published by producer before running the upgrade playbook. Details can be found in "Tests to make sure no data loss between producer and consumer during scale up and scale down" Section in the below link.

```
https://github.com/SupriyaPrasad/Scaling-Kafka-nodes.git
```

### How Rolling Upgrade Is Done?

Every node mentioned under [kafka-upgrade-nodes] in the ansible-kafka-upgrade/hosts file is picked one at a time.

```
sfuser@sfuser-virtual-machine:~/ansible-kafka-upgrade$ cat upgrade.yml 

- hosts: kafka-upgrade-nodes 
  roles:
    - role: ansible-kafka-upgrade
  sudo: yes
  serial: 1
```

Make the following changes in host files:

 1. Replace \<username\> below with the user name on the destination machines where the modules would be installed and configured by ansible. 
 2. The IP addresses have to be replaced with your node ips.

```
sfuser@sfuser-virtual-machine:~/ansible-kafka-upgrade$ cat hosts

[zookeepers]
192.168.0.168 zookeeper_id=1 ansible_ssh_user=<username>

[kafka-nodes]
192.168.0.165 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_ssh_user=<username>
192.168.0.164 kafka_broker_id=2 kafka_hostname=kafka-2 ansible_ssh_user=<username>
192.168.0.166 kafka_broker_id=3 kafka_hostname=kafka-consumer ansible_ssh_user=<username>

[kafka-upgrade-nodes]
192.168.0.164 kafka_broker_id=2 kafka_hostname=kafka-2 ansible_ssh_user=<username>
192.168.0.165 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_ssh_user=<username>
```

Please note that the kafka_broker_id=3 is used to run the consumer script and doesn't participate in upgrade

```
bin/kafka-console-consumer.sh --zookeeper 192.168.0.170:2181 --topic new
```

The existing kafka version is removed from each host. This is achieved by adding the dependecies.

```
sfuser@sfuser-virtual-machine:~/ansible-kafka-upgrade$ cat roles/ansible-kafka-upgrade/meta/main.yml 
---
dependencies: [ansible-kafka-remove]
```

The upgrade version is mentioned in defaults.

```
sfuser@sfuser-virtual-machine:~/ansible-kafka-upgrade$ cat roles/ansible-kafka-upgrade/defaults/main.yml 
apt_cache_timeout: 3600
kafka:
  version: "0.8.2.1"
  scala_version: "2.10"
  mirror: "http://mirrors.koehn.com/apache"
  download_dir: "/tmp"
  install_dir: "/usr/local"
  data_dir: "/usr/local/var/run/kafka/data"
  server_properties: "real-server.properties"

# This does not have to be every Zookeeper host, but the more the better
# by default, we assume this is run at the same time as Zookeeper provisioning
zk_hosts: "{{ groups['zookeepers'] }}" # This does not have to be every Zookeeper host
zk_client_port: 2181
installed_java_version: "java-7-oracle"
```

### Running the playbook

```
sfuser@sfuser-virtual-machine:~/ansible-kafka-upgrade$ ansible-playbook -i hosts upgrade.yml -K
```

## Verify Upgrade is Successful

Login to the all kafka servers and check if the version(kafka_2.10-0.8.2.1) is updated. The kafka jar version should be replaced with the new one and the previous version should be deleted.

```
sfuser@sfuser-virtual-machine:/usr/local$ ls k*
kafka:
bin  config  libs  LICENSE  logs  NOTICE

kafka_2.10-0.8.2.1:
bin  config  libs  LICENSE  logs  NOTICE
```

Verify the consumer continues to get all the messages published by producer during and after the upgrade.Details can be found in "Tests to make sure no data loss between producer and consumer during scale up and scale down" Section in the below link.

```
https://github.com/SupriyaPrasad/Scaling-Kafka-nodes
```
