# Datavisor Onsite Deployment Practice

-----

### 																																		--Author: Yixin Huang

## 1. CONNECT TO THE SERVER

[Introduction slides](https://docs.google.com/presentation/d/1ZjRX0jGhtvvFUkofyI_mGGaSUSlU6-eDZJmGBpMImAo/edit?pli=1#slide=id.g82da3ea189_1_8)

**Available instances**:

*We have already put the package in /packages directory of onsite1 and onsite4 servers.*

- **Group 1**:
  - 10.201.33.67   onsite1
  - 10.201.34.192 onsite2
  - 10.201.42.238 onsite3
- **Group 2**:
  - 10.201.43.34   onsite4
  - 10.201.44.57   onsite5
  - 10.201.45.159 onsite6

For me, I ask for help from ***Chang Liu*** to add my public ssh key to the ***onsite1*** Then I can connect this node by typing

`ssh datavisor@10.201.33.67`

## 2. Make sure the necessary file existed

After connected to onsite1, check whether the necessary file exists on the following path:

 `/packages/datavisor-onsite.tar.gz`

compres the file to get the new file: onsite-ansible-docker

`tar -xvf datavisor-onsite.tar.gz`

**I skip the Step 2 in the README.md since I already have the "datavisor" user**

## 3. Install Ansible

*Keep operating in onsite1 env.*

**Some Operations may need sudo**

```shell
$ tar -zxvf datavisor-onsite.tar.gz
$ cd onsite-ansible-docker/
$ tar -xzvf ansible-centos7.4.1708.tar.gz   # it works now
$ sudo yum install ansible # do this to install ansible
$ cd ansible-centos7.4.1708 # skip & ignore this line due to the broken existed file
$ sudo ./install.sh # skip & ignore this line due to the broken existed file
$ ansible --version
ansible 2.9.6 # 2020.04.19
  config file = /packages/onsite-ansible-docker/ansible.cfg
  configured module search path = [u'/home/datavisor/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Oct 30 2018, 23:45:53) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

**due to the space of /packages is not enough, we need to set the directory "/datavisor" under the /data**

## 4. Configure the environment

```shell
# for me, the onsite1 is the Center Control Machine, so I pass IPs of onsite2 and onsite3 to fpi arguments

$ ./generate_base_ha_config.py -d /data/datavisor -i 10.201.33.67,10.201.34.192,10.201.42.238 -c 5 -m 60g -o base_config.ini --fpi 10.201.34.192,10.201.42.238
```

> All services will run with first ip. Spark slave, hadoop datanode, hbase regionserver and moniter node will run with all ips. We can change the topology manually after we generate the config file.

There would be a *base_config.ini* file generated in the current directory.

## 5. Bootstrap

**Chang** said:

>哦哦 这步 应该是想要保持，中控机的 cat /etc/hosts 里对应的" ip  hostname" ，和每台机器上的hostname 是否一致。
>因为有些组件，例如HDFS，会有这样的映射，hdfs-datanode，他会去的对应hostname 和 ip。
>
>如果实际操作 我们一般会把master机器 /etc/hosts 里，手动把所有机器都列进去
>ip1	datavisor-server0001
>ip2 datavisor-server0002
>ip3 datavisor-server0003
>.
>.
>.
>
>这样。
>也方便自己  ssh datavisor-server0002
>
>我们onsite training的环境里现在应该是这样哈：
>ip1	onsite1
>ip2 onsite2
>ip3 onsite3

`ansible-playbook playbooks/bootstrap.yml --private-key=~/.ssh/id_rsa

```shell
PLAY RECAP ***********************************************************************************************
10.201.33.67   : ok=33   changed=13   unreachable=0    failed=0    skipped=1    rescued=0    ignored=2
10.201.34.192  : ok=33   changed=13   unreachable=0    failed=0    skipped=1    rescued=0    ignored=3
10.201.42.238  : ok=33   changed=13   unreachable=0    failed=0    skipped=1    rescued=0    ignored=3
```

### Step 6 is same as README.md file, which will run about 15 mins.

### Step 7 Integration tests (optional) cannot execute successfully (ignore this step.)

### Step 8 may fail due to the connection/network error, you can try it later. 

<font size=5 color=salmon>Please Remember to clean the env when you finish your practice by running the following commands</font>

```shell
$ ansible-playbook playbooks/clean_all.yml
Are you sure you want to clean everything? Answer with 'YES' [NO]: YES

$ sudo yum remove ansible
```

g