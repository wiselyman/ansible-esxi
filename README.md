### 使用Ansible自动创建ESXI虚拟机

#### 1.前置条件
- 操作机已安装`ansible`、`jinja2`，`sshpass`;
- `192.168.1.155`已安装`ESXI`。

#### 2.代码结构

```
wangyunfeideMBP:ansible-esxi wangyunfei$ tree
.
├── ansible.cfg
├── group_vars
│   └── all.yml
├── inventory.cfg
├── ssh.yml
├── templates
│   └── ifcfg-ens192.j2
└── vm.yml
```

#### 3.ansible配置
`ansible.cfg`

```
[defaults]
host_key_checking = False
```
此句的目的是让连接远程主机的时候不提示:

```
The authenticity of host '192.168.1.* (192.168.1.*)' can't be established.
ECDSA key fingerprint is SHA256:ONOCcjMG9+ZSLpiKPCOT5hkHjVujFusUBbQNtr9v4uY.
Are you sure you want to continue connecting (yes/no)? 
```

#### 4.变量
`group_vars/all.yml`

```json
---
esx_hostname: localhost.localdomain
esx_ipaddr: 192.168.1.155 
esx_user: root　
esx_password: zzzzzzz
datastore_name: HDD
iso_path: HDD/ISO/CentOS-7-x86_64-DVD-1708_Wisely-dhcp.iso
vm_state: powered_on
disk_size: 100
mem_size: 4096
```
这里的`iso_path`是可以自动安装的镜像，在[定制自动安装的CentOS7镜像](http://www.wisely.top/2017/11/16/centos7-auto-install/)中有制作方式。这里特别说明，因我司购买的的序列号不支持直接复制虚拟机模板，所以只能使用光盘安装。

#### 5.网络模板
`templates/ifcfg-ens192.j2`

```
{% if tag == 'k-node1' %}
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME=eth0
DEVICE=ens192
ONBOOT=yes
IPADDR="192.168.1.136"
PREFIX="24"
GATEWAY="192.168.1.1"
DNS1="114.114.114.114"
{% endif %}

{% if tag == 'k-node2' %}
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME=eth0
DEVICE=ens192
ONBOOT=yes
IPADDR="192.168.1.137"
PREFIX="24"
GATEWAY="192.168.1.1"
DNS1="114.114.114.114"
{% endif %}
```
这个文件是`jinja2`模板，根据不同的虚拟机名称，配置不同的静态ip。

#### 6.创建虚拟机并修改IP

```json
---
- hosts: 127.0.0.1
  connection: local
  user: root
  sudo: false
  gather_facts: false
  serial: 1
  tasks:
    - name: Deploy guest from ISO
      vsphere_guest:
        vcenter_hostname: "{{ esx_ipaddr }}"
        username: root
        password: zzzzzzz
        validate_certs: no
        guest: "{{ item }}"
        state: "{{ vm_state }}"
        vm_extra_config:
          vcpu.hotadd: yes
          mem.hotadd:  yes
          notes: This is a test VM
        vm_disk:
          disk1:
            size_gb: "{{ disk_size }}"
            type: thin
            datastore: "{{ datastore_name }}"
        vm_nic:
          nic1:
            type: vmxnet3
            network: VM Network
            network_type: standard
        vm_hardware:
          memory_mb: "{{ mem_size }}"
          num_cpus: 2
          osid: centos7_64Guest
          scsi: paravirtual
          vm_cdrom:
            type: "iso"
            iso_path: "{{ iso_path }}"
        esxi:
          datacenter: ha-datacenter
          hostname: "{{ esx_hostname  }}"
      with_items:
        - "k-node1"
        - "k-node2"


    - name: Get Facts of vms  #获得虚拟机的信息
      vsphere_guest:
        vcenter_hostname: "{{ esx_ipaddr }}"
        username: root
        password: zzzzzzz
        validate_certs: no
        guest: "{{ item }}"
        vmware_guest_facts: yes
        esxi:
          datacenter: ha-datacenter
          hostname: "{{ esx_hostname  }}"
      register: vmguest_facts
      until: vmguest_facts.ansible_facts.hw_eth0.ipaddresses
      retries: 30
      delay: 10
      with_items:
        - "k-node1"
        - "k-node2"
 
    - name: Add the IP address of the VM to Memory Inventory #动态添加虚拟机ip到内存中的Inventory
      add_host:
        hostname: "{{ item.ansible_facts.hw_eth0.ipaddresses[0] }}"
        ansible_ssh_user: root
        ansible_ssh_pass: zzzzzz
        tag: "{{ item.ansible_facts.hw_name }}"
      with_items: "{{ vmguest_facts.results }}"

- hosts: all
  tasks:
    - name: changing IP of vms # 修改虚拟机自动配置的ip到我们需要的ip
      template: src=templates/ifcfg-ens192.j2 dest=/etc/sysconfig/network-scripts/ifcfg-ens192

    - name: Restart vms # 重启虚拟机
      become: yes
      shell: sleep 2 && /sbin/shutdown -r now "Ansible system package upgraded"
      async: 1
      poll: 0

```

#### 7.当前操作机免密登录虚拟机

生成钥匙对: `ssh-keygen`

`inventory.cfg`

```
[nodes]
192.168.1.136 ansible_ssh_user=root ansible_ssh_pass=zzzzzz
192.168.1.137 ansible_ssh_user=root ansible_ssh_pass=zzzzzz
```

`vm.yml`

```json
- hosts: nodes
  tasks:
    - name: Creates ssh directory
      file: path=/root/.ssh state=directory

    - name: Copy authorized_keys #复制本机公钥到指定机器
      template: src=/Users/wangyunfei/.ssh/id_rsa.pub dest=/root/.ssh/authorized_keys
```

#### 8.执行
- 创建虚拟机并修改IP

`ansible-playbook vm.yml`

- 当前操作机免密登录虚拟机

`ansible-playbook -i inventory.cfg ssh.yml`


