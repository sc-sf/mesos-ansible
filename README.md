Setting up Mesos and Zookeeper with Ansible


Introduction

Mesos is a cluster management software. It is often considered as an OS kernel for distributed computing. An OS kernel on a single computer manages hardware resources, i.e., cpu, disk storage and memory etc. Applications running on an OS communicate with the kernel for such resources. Mesos pretty much does exactly the same thing. However, instead of a kernel working on compute resources on a single machine, what underpins this mesos kernel is a giant pool of computers, or all the nodes in a datacenter. Indeed mesos calls itself a datacenter operating system, or DCOS. 

Besides Mesos, we are also installing Zookeeper. With help of Zookeeper, multiple master servers can participate in a cluster to provide high availability. Zookeeper calls this group of servers an ensemble, typically consisting an odd number of nodes, at least 3, which is what we have in here. At any given time, there's only one active master server in the ensemble while the rest are in standby mode. The active master server is called the leader through a process called leader selection in Zookeeper. This process is based on a consensus algorithm.  


Prerequisites and assumptions

All the ansible playbooks and config files can be downloaded with this command

      git clone git://github.com/sc-sf/mesos-ansible.git

This git repo installs and configures Mesos and Zookeeper, which serves as a prerequisite for another git repo dealing with "How to configure hdfs and mapreduce on mesos with ansible". At the bottom of this guide, you can see the directory layout of this repo. You might want to copy the whole layout into a separate text file so that you can refer to it more easily. Of the following items, you will need to download mesos-0.21.0-1.0.rpm and zookeeper-3.4.6.tar.gz independently.

* zookeeper-3.4.6.tar.gz
* mesos-0.21.0-1.0.rpm
* centos 6.5
* hostnames can be resovled by either dns or hosts file
* ansible version 1.7.2 or above
* the "ansible" user is already created on all machines
* three or five master servers and three or more slave servers

mesos-0.21.0-1.0.rpm can be installed using yum and then grab a copy in yum's cache, which is located in /var/cache/yum/. Please note the major version of mesos being specified explicitly. If you don't, yum will download the current latest, which is not the same version this repo is based on.

      sudo yum install mesos-0.21.0


zookeeper-3.4.6.tar.gz can be found here
     
      https://archive.apache.org/dist/zookeeper/stable/ 


Getting started

At the high level, mesos consists of a set of master servers and a set of slave servers. The tasks in this playbook is broken into three parts. The first part deals with configurations that are common to all the nodes in the cluster whether they are master or slave. The second part is slave specific while the last deals with master only.


Part 1: configurations that are common to all nodes

Copy the package over to each machine and then install it. Notice the src parameter would look for the package in files directory of the same role the playbook is in. After Mesos is installed, we need to first specify the zookeeper url showing the list of nodes in zookeeper ensemble, which are the 3 mesos master servers. Because Zookeeper handles the fault tolerance of the Mesos master servers by keeping track of their states and be in charge of the leader selection, it knows best which master is active and leading. So whoever needs to communicate with the active master, it has to ask zookeeper for it. Indeed, this is called the service discovery and is also part of what Zookeeper does. The zookeeper url is put into the "zk" file in this format: 

      zk://172.16.1.11:2181,172.16.1.12:2181,172.16.1.13:2181/mesos

If you look at the ansible's site-wide variables file called "all" in group_vars directory, you can see how the list of IP addresses of the masters are generated based on the masters group in the inventory file called "production". The result of the list is put into the zk_ensemble_url variable. Notice the leading_master variable is dynamically determined by picking the first hostname in the masters group. 


      # ---------- to all nodes in the cluster--------------------------------------------------------

      - name: copy mesos-0.21.0-1.0.rpm 
        copy: src={{mesos_rpm}} dest=/tmp

      - name: install mesos 
        yum: pkg=/tmp/{{mesos_rpm}} state=installed

      - name: define the IPs of mesos master service that zookeeper coordinates
        template: src=zk.j2 dest=/etc/mesos/zk owner=0 group=0 mode=644


Part 2: Configurations that are slave specific

By using ansible's conditional when statement, all the tasks are only applied to hosts in the slaves group, as the builtin inventory_hostname variable would resolve to the hostname of the current remote machine being worked on. 

There are more than one way to pass settings or arguments to mesos. Here we pass the arguments in files in the /etc/mesos-slave directory where mesos would pick up right before each time its daemon starts. The name of the argument is the name of the file. So we need to define 3 arguments: hostname, ip and resources. While the hostname and ip arguments are pretty self-explained, the resources argument is a bit more convoluted. Slaves report to the master what resources that they have and the master, based on the reported resources, makes resource offers to applications running on mesos. In between the resources offering and acceptance, there are different pluggable policies that can be configured to manipulate how the resources be distributed and utilized. If the resources argument is not explicitly defined, the slave would report what cpu, disk, and memory it actually has. But the flexibility and power behind this definable resources argument is that it can be overstated or understated than what the actual resources the slaves have, making resource sharing more granular and fine grained. 
   
Next is the upstart init script. By default mesos package already includes one mesos-master.conf init script and one mesos-slave.conf init script that are ready to get inited. So we don't really need to do anything, unless you want to customize some environment variables that the mesos daemon would need. And this is exactly what we are trying to do here. If you take a look at the mesos-slave.conf.j2 template you can see there are some environment variables being defined before the daemon gets started. 

The last ansible task is to remove the mesos-master.conf file as you probably don't want the master daemon running  on a slave. Here we use ansible's shell module. (The ansible file module with state=absent would have the same effect.) 


      # ---------- slaves specific --------------------------------------------------------

      - name: specify the hostname of mesos
        template: src=hostname.j2 dest=/etc/mesos-slave/hostname owner=0 group=0 mode=644
        when: inventory_hostname in groups['slaves']

      - name: specify the ip address of mesos... start mesos-slave
        template: src=ip.j2 dest=/etc/mesos-slave/ip owner=0 group=0 mode=644
        when: inventory_hostname in groups['slaves']

      - name: specify the usable resources per slave - the numbers can overstate
        template: src=resources.j2 dest=/etc/mesos-slave/resources owner=0 group=0 mode=644
        when: inventory_hostname in groups['slaves']

      - name: copy mesos slave init script
        template: src=mesos-slave.conf.j2 dest=/etc/init/mesos-slave.conf owner=0 group=0 mode=644
        when: inventory_hostname in groups['slaves']
        notify:
          - start mesos-slave

      - name: disable slave... enable=no of service module doesn't work 
        shell: rm -fr /etc/init/mesos-master.conf
        when: inventory_hostname in groups['slaves']


Part 3: Configuring the masters

Mesos needs to know how many replicas of its replicated_log should be created. And the number should be equal to the quorum of the ensemble. The ensemble, in our example consists of three masters so the quorum would be 2. If you look the quorum.j2 template, you can see that the value gets determined dynamically based the number of master servers in the masters group in the inventory file. When there are three hosts in the masters group the quorum is 2. Otherwise it will assume the ensemble of five members and the quorum would be 3. Remember quorum is the majority vote of the ensemble. 2 out of 3 is a majority; 3 out of 5 is also a majority. The rest of the settings (hostname, ip and mesos-master.conf.j2) have the same reasoning as the slaves. 

As part of configuring the master servers, we also need to setup the zookeeper. Zookeeper is installed in the /etc directory along with a symlink creation. We then need to uniquely define an ID for each zookeeper server in the /etc/zookeeper/myid file. Here the ansible's host specific variables are utilized. Inside the host_vars directory contains host specific files each containing its unique ID number. 

In addition, each ID is required to explicitly map to the IP address respectively in the zoo.cfg file as follows:

      server.1={{ master1 }}:2888:3888
      server.2={{ master2 }}:2888:3888
      server.3={{ master3 }}:2888:3888

In an ensemble, there's only one leader and the rest of the servers are called followers. The first port 2888 is used by the followers when communicating with the leader. The second port 3888 is used only during the leader election process. Next, we need to come up with our own zookeeper init script as by default the zookeeper package doesn't have one. The last ansible task we need to run is of course remove the mesos-slave init script so there won't be a chance of running the slave daemon on master server.


      # ---------- masters specific --------------------------------------------------------

      - name: specify number of replicas of replicated_log; the quorum value
        template: src=quorum.j2 dest=/etc/mesos-master/quorum owner=0 group=0 mode=644
        when: inventory_hostname in groups['masters']

      - name: specify the hostname of the master
        template: src=hostname.j2 dest=/etc/mesos-master/hostname owner=0 group=0 mode=644
        when: inventory_hostname in groups['masters']

      - name: specify the ip address of the master
        template: src=ip.j2 dest=/etc/mesos-master/ip owner=0 group=0 mode=644
        when: inventory_hostname in groups['masters']

      - name: copy mesos master init script
        template: src=mesos-master.conf.j2 dest=/etc/init/mesos-master.conf owner=0 group=0 mode=644
        when: inventory_hostname in groups['masters']
        notify:
          - start mesos-master

      - name: copy zookeeper-3.4.6.tar.gz over to masters only
        copy: src={{zookeeper_gz}} owner=0 group=0 dest=/tmp
        when: inventory_hostname in groups['masters']

      - name: untar zookeeper-3.4.6.tar.gz 
        # unarchive module doesn't change 1000:1000 ownership. But zookeeper doesn't seem to bother with it
        unarchive: src=/tmp/{{zookeeper_gz}} copy=no dest=/etc owner=0 group=0
        when: inventory_hostname in groups['masters']

      #- name: fix owner permission thing because unarchive above ignores it
      #  file: path=/etc/zookeeper state=directory group=0 owner=0 recurse=yes

      - name: create symlink 
        file: src={{zookeeper}} dest=/etc/zookeeper owner=0 group=0 state=link
        when: inventory_hostname in groups['masters']

      - name: set the unique id for each zookeeper server
        copy: content={{zk_myid}} dest=/etc/zookeeper/myid
        when: inventory_hostname in groups['masters']

      - name: map the id to the ip address of each master in zoo.cfg
        template: src=zoo_cfg.j2 dest=/etc/zookeeper/conf/zoo.cfg owner=0 group=0 mode=644
        when: inventory_hostname in groups['masters']

      - name: create zookeeper init script
        template: src=zookeeper.j2 dest=/etc/init.d/zookeeper owner=0 group=0 mode=755
        when: inventory_hostname in groups['masters']

      - name: add zookeeper to chkconfig and start zookeeper
        shell: chkconfig --add zookeeper
        when: inventory_hostname in groups['masters']
        notify:
          - start zookeeper

      - name: disable mesos-slave.conf... enable=no of service module doesn't work
        shell: rm -fr /etc/init/mesos-slave.conf
        when: inventory_hostname in groups['masters']





