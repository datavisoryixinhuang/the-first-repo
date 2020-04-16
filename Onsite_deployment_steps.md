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

`ansible-playbook playbooks/bootstrap.yml --private-key=/path/of/your/private/key` (1)

Then **Chaoqun Huang** found that the ansible may not be installed correctly. Then we re-install it by running the following command

```shell
# code to install
$ yum install ansible

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
PLAY RECAP *****************************************************************************
10.201.33.67   : ok=5    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=3
10.201.34.192  : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
10.201.42.238  : ok=5    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=3
```

**To be fixed...**



