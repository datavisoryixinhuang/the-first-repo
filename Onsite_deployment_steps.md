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

Keep operating in onsite1 env.

**Some Operations may need sudo**

```shell
$ tar -zxvf datavisor-onsite.tar.gz
$ cd onsite-ansible-docker/
$ tar -xzvf ansible-centos7.4.1708.tar.gz   # it works now
$ cd ansible-centos7.4.1708
$ sudo ./install.sh
$ ansible --version
ansible 2.4.2.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/datavisor/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Oct 30 2018, 23:45:53) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

## 4. Configure the environment

```shell
# configure the env.
$ ./generate_base_ha_config.py --help
Usage: generate_base_ha_config.py [options]

Generate ansible base config

Options:
  -h, --help            show this help message and exit
  -d DATAVISOR_DIR, --datavisor-dir=DATAVISOR_DIR
                        datavisor dir for data, docker root and scripts
  -t CLIENT_NAME, --client-name=CLIENT_NAME
                        client company name used to differ onsite deployment
                        configs
  -i IPS, --ips=IPS     ip list, seperated by comma. First ip will be the
                        master
  -m WORKER_MEMORY, --worker-memory=WORKER_MEMORY
                        set how much total memory workers have to give
                        executors
  -c WORKER_CORES, --worker-cores=WORKER_CORES
                        the number of cores to use on each spark slave node
  -n NUMBER_WORKERS, --number-workers=NUMBER_WORKERS
                        [Deprecated]the number of worker instances on each
                        spark worker node
  -o OUTPUT, --output=OUTPUT
                        output file
  --mode=MODE           deployment mode, binary|docker
  --user=USER           user to deploy
  --fpi=FPI             feature platform ips
  
# for me, the onsite1 is the Center Control Machine, so I pass IPs of onsite2 and onsite3 to fpi arguments

$ ./generate_base_ha_config.py -d ./datavisor -i 10.201.33.67,10.201.34.192,10.201.42.238 -c 5 -m 60g -o base_config.ini --fpi 10.201.34.192,10.201.42.238
```

> All services will run with first ip. Spark slave, hadoop datanode, hbase regionserver and moniter node will run with all ips. We can change the topology manually after we generate the config file.

There would be a *base_config.ini* file generated in the current directory.

## 5. Bootstrap

I met the some errors when trying to execute the 

`ansible-playbook playbooks/bootstrap.yml --private-key=~/.ssh/id_rsa` (1)

Then **Chaoqun Huang** found that the ansible may not be installed correctly. Then we re-install it by running the following command

```shell

$ ansible --version
ansible 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/datavisor/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Oct 30 2018, 23:45:53) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

However, after re-install this, I still met error on running code (1), here is the part of outcome:

```shell
fatal: [10.201.33.67]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: 'ansible.vars.hostvars.HostVarsVars object' has no attribute 'ansible_hostname'\n\nThe error appears to be in '/packages/onsite-ansible-docker/roles/bootstrap/tasks/main.yml': line 55, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: Remove 127.0.0.1 mapping to hostname\n  ^ here\n"}
fatal: [10.201.42.238]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: 'ansible.vars.hostvars.HostVarsVars object' has no attribute 'ansible_hostname'\n\nThe error appears to be in '/packages/onsite-ansible-docker/roles/bootstrap/tasks/main.yml': line 55, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: Remove 127.0.0.1 mapping to hostname\n  ^ here\n"}

PLAY RECAP *****************************************************************************
10.201.33.67   : ok=5    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=3
10.201.34.192  : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
10.201.42.238  : ok=5    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=3
```

**According form Chang Liu, it is the hosts file that causes this error, so I checked the host file as following on onsite1**

`vim /etc/hosts`

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.201.33.67 onsite1
10.201.34.192 onsite2
10.201.42.238 onsite3
```

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

So, after I tried to comment first 2 lines, it still failed with same error. Then I just deteled the first two line, and this is what looks like for /etc/hosts

```
10.201.33.67 onsite1
10.201.34.192 onsite2
10.201.42.238 onsite3
```

*I need to mentioned that I still met some errors on onsite2 machine, so I ssh to it form onsite1 and found there is a file that obviously should not be there under /datavisor. After I deleted it and back to onsite1, re-executed the command, it pops up some different errors*

`ansible-playbook playbooks/bootstrap.yml --private-key=~/.ssh/id_rsa`

```shell
TASK [jdk8 : Set root directory] ************************************************************************************
ok: [10.201.33.67]
ok: [10.201.34.192]
ok: [10.201.42.238]
Friday 17 April 2020  07:52:42 +0000 (0:00:00.104)       0:00:21.599 **********

TASK [jdk8 : Check if jdk exists] ***********************************************************************************
ok: [10.201.33.67]
ok: [10.201.42.238]
ok: [10.201.34.192]
Friday 17 April 2020  07:52:44 +0000 (0:00:01.211)       0:00:22.811 **********

TASK [jdk8 : Ensure jdk base dir] ***********************************************************************************
changed: [10.201.33.67]
changed: [10.201.34.192]
changed: [10.201.42.238]
Friday 17 April 2020  07:52:45 +0000 (0:00:01.054)       0:00:23.866 **********

TASK [jdk8 : Copy jdk to hosts] *************************************************************************************
fatal: [10.201.33.67]: FAILED! => {"changed": false, "msg": "Failed to find handler for \"/home/datavisor/.ansible/tmp/ansible-tmp-1587109965.26-68394309318502/source\". Make sure the required command to extract the file is installed. Command \"unzip\" not found. Command \"/usr/bin/gtar\" could not handle archive."}
fatal: [10.201.34.192]: FAILED! => {"changed": false, "msg": "Failed to find handler for \"/home/datavisor/.ansible/tmp/ansible-tmp-1587109965.29-182728417559723/source\". Make sure the required command to extract the file is installed. Command \"unzip\" not found. Command \"/usr/bin/gtar\" could not handle archive."}
fatal: [10.201.42.238]: FAILED! => {"changed": false, "msg": "Failed to find handler for \"/home/datavisor/.ansible/tmp/ansible-tmp-1587109965.32-147762919821206/source\". Make sure the required command to extract the file is installed. Command \"unzip\" not found. Command \"/usr/bin/gtar\" could not handle archive."}

PLAY RECAP ********************************************************************************************
10.201.33.67    : ok=17   changed=6    unreachable=0    failed=1    skipped=1    rescued=0    ignored=3
10.201.34.192   : ok=17   changed=6    unreachable=0    failed=1    skipped=1    rescued=0    ignored=3
10.201.42.238   : ok=17   changed=6    unreachable=0    failed=1    skipped=1    rescued=0    ignored=3

Friday 17 April 2020  07:52:49 +0000 (0:00:04.371)       0:00:28.237 **********
===============================================================================
jdk8 : Copy jdk to hosts ---------------------------------------------------------------- 4.37s
bootstrap : Remove 127.0.0.1 mapping to hostname ---------------------------------------- 3.24s
bootstrap : Remove ipv6 ::1 mapping to hostname ----------------------------------------- 3.04s
Gathering Facts ------------------------------------------------------------------------- 2.27s
bootstrap : Disable firewalld ----------------------------------------------------------- 1.49s
bootstrap : Disable SElinux ------------------------------------------------------------- 1.41s
bootstrap : Create user ----------------------------------------------------------------- 1.37s
bootstrap : Create datavisor dir -------------------------------------------------------- 1.25s
bootstrap : Create group ---------------------------------------------------------------- 1.23s
jdk8 : Check if jdk exists -------------------------------------------------------------- 1.21s
bootstrap : Set DV_DIR environment parameter -------------------------------------------- 1.21s
bootstrap : Update /etc/security/limits.conf -------------------------------------------- 1.19s
bootstrap : Disable Enforcement --------------------------------------------------------- 1.17s
bootstrap : Disable iptables ------------------------------------------------------------ 1.16s
jdk8 : Ensure jdk base dir -------------------------------------------------------------- 1.05s
bootstrap : Get hostname by shell ------------------------------------------------------- 1.03s
jdk8 : Get root directory of the package ------------------------------------------------ 0.32s
jdk8 : Set root directory --------------------------------------------------------------- 0.10s
bootstrap : If hostname contains _, will rewrite hostname ------------------------------- 0.10s
Playbook run took 0 days, 0 hours, 0 minutes, 28 seconds
```

As the error message says that  <font color=darksalmon>Command \"unzip\" not found.</font>, I install the **unzip** by:

```shell
$ sudo yum install unzip
$ which gtar
/usr/bin/gtar
$ which unzip
/usr/bin/unzip
# however, I still met same error :(
```



