![](https://hackmd.io/_uploads/HyLzPAKsh.png)
![](https://hackmd.io/_uploads/ryvDHx5j2.png)
![](https://hackmd.io/_uploads/Byi6vxqs2.png)
![](https://hackmd.io/_uploads/S1gJaSooh.png)
![](https://hackmd.io/_uploads/SkGa683h3.png)
# Ansible
## Installation
```bash=
sudo dnf install ansible-navigator
ansible-navigator --version
podman login utility.lab.example.com
ansible-navigator images
```
## Inventory
> An inventory defines a collection of hosts that Ansible manages. 

### Defining
```ini=
[usa]
washington1.example.com
washington2.example.com
washington[1:2].example.com

[canada]
ontario01.example.com
ontario02.example.com
ontario[01:02].example.com


[north-america:children] # nested
canada
usa
```
> Two host groups always exist:
> 1. all
> 2. ungrouped

### Verifying
```bash=
[user@controlnode ~]$ ansible-navigator inventory -m stdout --host washington1.example.com --graph 
```

## Configuration
```bash=
[defaults]
inventory = ./inventory
remote_user = user # 不指定用control node
ask_pass = false

[privilege_escalation] # 被管理主機
become = true
become_method = sudo
become_user = root
become_ask_pass = false
```

### ansible-navigator.yml
```yaml=
---
ansible-navigator:
  execution-environment:
    image: utility.lab.example.com/ee-supported-rhel8:latest
    pull:
      policy: missing
  playbook-artifact:
    enable: false
```
```bash=
ansible-navigator images
```
## Playbook
### example
![](https://hackmd.io/_uploads/B1iXtnkah.png)

> 當然你也可以在不同的play中提權
![](https://hackmd.io/_uploads/SkCVK31ph.png)
### Modules
- Files
	- ansible.builtin.copy
	- ansible.builtin.file
	- ansible.builtin.lineinfile
- Software
	- ansible.builtin.dnf
	- ansible.builtin.apt
- System
	- ansible.posix.firewalld
	- ansible.builtin.reboot
	- ansible.builtin.service
	- ansible.builtin.user
	- ansible.builtin.command # 不建議
	- ansible.builtin.shell # echo
- Net tools
	- ansible.builtin.get_url
	- ansible.builtin.uri
```bash
# 不知道怎麼用的時候 有兩個救命仙丹
ansible-doc ping # adhoc command module 也找的到!
ansible-navigator doc -l # 用於執行環境
ansible-navigator doc {module} # 用於執行環境
ansible-navigator doc --search {module_fuzz} # 用於執行環境
ansible-navigator doc ansible.builtin.dnf -m stdout
```
> Note:
![](https://hackmd.io/_uploads/SJuoao7p2.png)
![](https://hackmd.io/_uploads/Hk3iTj762.png)
![](https://hackmd.io/_uploads/SJlnTsmpn.png)
![](https://hackmd.io/_uploads/ryShpjQT3.png)
![](https://hackmd.io/_uploads/SJcIJp76n.png)


## Variable
### Define Var
#### in play
```yaml=
- hosts: all
  vars:
    user: jack
	home: /home/jack
  vars_files: # 吃檔案
    - vars/users.yml
```
#### in inventory
```ini=
[servers]
demo.example.com ansible_user=joe # 直接給這個host

[server_group]
demo1.example.com
demo2.example.com

[server_group:var] # 對group
user=joe
```

#### Using Directories
assume you have the inventory below
```ini=
[datacenter1]
demo1.example.com
demo2.example.com
[datacenter2]
demo3.example.com
demo4.example.com
[datacenters:children]
datacenter1
datacenter2
```
```bash=
[admin@station project]$ cat ~/project/group_vars/datacenters
package: httpd
[admin@station project]$ cat ~/project/group_vars/datacenter1
package: httpd
[admin@station project]$ cat ~/project/group_vars/datacenter2
package: apache
```
> 你可以用group_vars的方式給，但要注意資料夾階層，像是下面這樣
```bash=
project
├── ansible.cfg
├── group_vars
│ ├── datacenters
│ ├── datacenters1
│ └── datacenters2
├── host_vars
│ ├── demo1.example.com
│ ├── demo2.example.com
│ ├── demo3.example.com
│ └── demo4.example.com
├── inventory
└── playbook.yml
```
#### via cli (overriding)
```bash=
[user@demo ~]$ ansible-navigator run main.yml -e "package=apache"
```

### Capaturing output to variable
![](https://hackmd.io/_uploads/SJddLhQ6h.png)

### Secrets
#### CRUD
```bash=
ansible-vault create secret.yml # 手動輸入密碼
ansible-vault create --vault-password-file=vault-pass secret.yml  #帶密碼檔
ansible-vault view secret.yml # 看
ansible-vault edit secret.yml # 編輯
```
#### Existing file
```bash=
ansible-vault encrypt secret1.yml secret2.yml # 加密
ansible-vault decrypt secret1.yml --output=secret1-decrypted.yml # 解密
ansible-vault rekey secret.yml # 重設密碼
```

#### 解密playbook via prompt
```bash=
ansible-navigator run -m stdout --playbook-artifact-enable false site.yml --vault-id @prompt
```

#### 解密playbook via pwfile
```bash=
echo "yourfuckingpassword" > vault-pw-file
ansible-navigator run -m stdout --playbook-artifact-enable false site.yml --vault-password-file=vault-pw-file
```

### Facts
Example
```yaml=
- name: Fact dump
  hosts: all
  tasks:
    - name: Print all facts
      ansible.builtin.debug:
        var: ansible_facts
```
![](https://hackmd.io/_uploads/rJyWj2Q6h.png)
```yaml=
---
# turn off
- name: This play does not automatically gather any facts
  hosts: large_datacenter
  gather_facts: no

# 隨時再把他打開
- name: This play does not automatically gather any facts
  hosts: large_datacenter
  tasks:
    - name: Manually gather facts
      ansible.builtin.setup:
	    gather_subset: # 我只要那些
		  - hardware
		  - !hardware # 我不要他
```
### Magic Variables
- hostvars: Contains the variables for managed hosts
- group_names: Lists all groups that the current managed host is in.
- groups: Lists all groups and hosts in the inventory.
- inventory_hostname: Contains the hostname for the current managed host as configured in the inventory

example
![](https://hackmd.io/_uploads/Sk8D3nXTh.png)
## Task Control
### Loop
```yaml=
- name: Postfix and Dovecot are running
  ansible.builtin.service:
    name: "{{ item }}" # for item in list
    state: started
    loop: # list
      - postfix
      - dovecot
```
### conditionally
```yaml=
tasks:
  - name: http packaged is installed
    ansible.builtin.dnf:
	  name: httpd
	when: true
```
> when conditionals example:
![](https://hackmd.io/_uploads/HktFfZH6h.png)
![](https://hackmd.io/_uploads/B1ccMWBp2.png)

### Combining Loops and Conditional Tasks
```yaml=
- name: install mariadb-server if enough space on root
  ansible.builtin.dnf:
    name: mariadb-server
    state: latest
  loop: "{{ ansible_facts['mounts'] }}"
  when: item['mount'] == "/" and item['size_available'] > 300000000
```
> 他要check每一個list中的item都符合條件才會做
### Implementing Handlers
> A task that runs only when another task **changes** the managed host.

```yaml=
tasks:
  - name: copy demo.example.conf configuration template
    ansible.builtin.template:
      src: /var/lib/templates/demo.example.conf.template
      dest: /etc/httpd/conf.d/demo.example.conf
    notify:
      - restart apache
handlers:
  - name: restart apache
    ansible.builtin.service:
	  name: httpd
	  state: restarted
```

### Handling Task Failure
#### Ignore
```yaml=
- name: latest version of notapkg is installed
  ansible.builtin.dnf:
    - name: notapkg
	  state: latest
  ignore_errors: yes # here
```
#### Forcing Execution of Handlers After Task Failure
> Notified handlers are called even if the play aborted because a later task failed
```yaml=
---
- hosts: all
force_handlers: yes
...ignore
```
#### Specifying Task Failure Conditions
```yaml=
tasks:
  - name: Run user creation script
    ansible.builtin.shell: /usr/local/bin/create_users.sh
    register: command_result
    failed_when: "'Password missing' in command_result.stdout" # 這不改變行為，只改變report
	
  - name: Report script failure
    ansible.builtin.fail: # fail的時候print message
	  msg: "The password is missing in the output"
    when: "'Password missing' in command_result.stdout"
```
#### Specifying Task Changed
```yaml=
- name: Validate httpd configuration
  ansible.builtin.command: httpd -t
  changed_when: false # 這裡有點像是說給一個bool決定task的狀態
  notify:
    - restart db # 利用changed_when來觸發handlers
...ignore
```
#### Ansible Blocks and Error Handling

| error handling | behavior |
| -------- | -------- |
| block     | main tasks to run  |
| rescue     | tasks to run if the tasks defined in the block clause fail  |
| always     | tasks that always run independently  |
```yaml=
tasks:
  - name: Upgrade DB
    block:
      - name: upgrade the database
        ansible.builtin.shell:
          cmd: /usr/local/lib/upgrade-database
    rescue:
      - name: revert the database upgrade
        ansible.builtin.shell:
        cmd: /usr/local/lib/revert-database
    always:
      - name: always restart the database
        ansible.builtin.service:
          name: mariadb
          state: restarted
```
> 理解上，我會覺得這個很像try catch block的架構

## Deploying Files on managed Host
File Modules:
![](https://hackmd.io/_uploads/HJOX1XSTn.png)
### Ensuring a File Exists
類似踏取指令
```yaml=
- name: Touch a file and set permissions
  ansible.builtin.file:
    path: /path/to/file
    owner: user1
    group: group1
    mode: 0640
    state: touch
```
### Modifying File Attributes
`ls -Z samba_file`
```yaml=
- name: SELinux type is set to samba_share_t
  ansible.builtin.file:
    path: /path/to/file
	setype: samba_share_t
```

### Copying and Editing Files

#### Copy
```yaml=
- name: Copy a file to managed hosts
  ansible.builtin.copy:
    src: file
	dest: /path/to/file
```
#### Fetch
```yaml=
- name: Retrieve SSH key from reference host
  ansible.builtin.fetch:
    src: "/home/{{ user }}/.ssh/id_rsa.pub
    dest: "files/keys/{{ user }}.pub"
```
#### Add line
```yaml=
- name: add a line of text to a file
  ansible.builtin.lineinfile:
    path: /path/to/file
    line: 'Add this line to the file'
    state: present
```
#### Add block
```yaml=
- name: add a block of text to a file
  ansible.builtin.blockinfile:
    path: /path/to/file
    block: |
    Add this block to the file
	  Add this block to the file
	  Add this block to the file
    state: present
```
#### Remove
```yaml=
- name: Make sure a file does not exist on managed hosts
  ansible.builtin.file:
    path: /path/to/file
    state: absent
```

#### Retrieving the Status of a File
```yaml=
- name: Verify the checksum of a file
  ansible.builtin.stat:
    path: /path/to/file
    # 以下checksum沒加的話, 就是檢查這個檔案在不在
    checksum_algorithm: md5 # 會有東西回來，可以用register接
```

#### Synchronizing Files
```yaml=
- name: synchronize local file to remote files
  ansible.posix.synchronize:
    src: file
	dest: /path/to/file
```
> symbolic 用法
```yaml=
- name: Ensure /etc/issue.net is a symlink to /etc/issue
  ansible.builtin.file:
    src: /etc/issue
    dest: /etc/issue.net
    state: link
    owner: root
    group: root
    force: yes
```

#### SELinux default
```yaml=
- name: SELinux file context is set to defaults
  ansible.builtin.file:
    path: /home/devops/users.txt
    seuser: _default
    serole: _default
    setype: _default
    selevel: _default
```

### Jinja2 templates
#### Deploying Jinja2 Templates
```yaml=
tasks:
  - name: template render
    ansible.builtin.template:
	  src: /tmp/j2-template.j2
	  dest: /tmp/dest/config-file.txt
```
### Controls
#### Loops
![](https://hackmd.io/_uploads/BJ1Au7H62.png)
#### If
![](https://hackmd.io/_uploads/BJTyK7Sah.png)
#### Varible Filters
```yaml=
{{ output | from_json }}
{{ output | to_json }}
{{ output | from_yaml }}
{{ output | to_yaml }}
```
## Role
### Creating a Role Skeleton

```shell=
[user@host ~]$ ansible-galaxy init
[user@host ~]$ tree roles/ roles/
└── motd
    ├── defaults # default variable
    │   └── main.yml
    ├── files
    ├── handlers
    ├── meta # dependencies
    │   └── main.yml
    ├── README.md
    ├── tasks  # tasks
    │   └── main.yml
    └── templates # jinja templates
        └── motd.j2
```
### Variable
> The value of any variable defined in a role's defaults directory is overwritten if that same variable is defined
例如說：
1. 你可以在你的play裡面去定義變數，使得roles的變數不一樣
![](https://hackmd.io/_uploads/r1ss2yC23.png)
2. 也可以在role的block中去改變
![](https://hackmd.io/_uploads/BJb521Rh2.png)

### Defining Role Dependencies
> roles允許你把其他的rule當成是dependency
> 寫在這邊 --> meta/main.yml

![](https://hackmd.io/_uploads/BkNfpJA33.png)

### External source
1. 定義你的requirements
![](https://hackmd.io/_uploads/HJmIhgAn2.png)
2. Run your galaxy to install (-p to specify roles path 不給可以用ansible.cfg)
![](https://hackmd.io/_uploads/ryuXEZA2n.png)
`ansible-galaxy role install -r roles/requirements.yml -p roles`
3. 或者你可以用ansible galaxy上面的roles
```shell=
ansible-galaxy search 'redis' --platforms EL
ansible-galaxy info geerlingguy.redis
cat roles/requirements.yml
```
```shell=
ansible-galaxy list
ansible-galaxy remove nginx
```
Summary: 

![](https://hackmd.io/_uploads/rJXLTxC23.png)

## Exam Dumps
### 請在 Workstation 上安裝 ansible 運作之必要套件
:::spoiler Reference
```yaml=
yum install vim
yum install ansible-core
yum install ansible-navigator
vim ~/.vimrc
autocmd FileType yaml setlocal ai ts=2
```
:::

### 組態 
及 ansible.cfg
```
建立靜態的 inventory 檔案 /home/student/ansible/inventory:
servera.lab.example.com 為 developer 群組的成員
serverb.lab.example.com 為 test 群組的成員
serverc.lab.example.com 和 serverd.lab.example.com 為production 群組的成員
servera.lab.example.com 為 balancer 群組的成員
production 群組為 webservers 的子群組
```
:::spoiler Reference
```yaml=
[developer]
servera.lab.example.com

[test]
serverb.lab.example.com

[production]
serverc.lab.example.com
serverd.lab.example.com

[balancer]
servera.lab.example.com

[webservers: children]
production
```
:::

```
建立設定檔 /home/student/ansible/ansible.cfg:
inventory 檔案路徑在 /home/student/ansible/inventory
預設的 content collections 路徑為/home/student/ansible/collections
預設的 roles 路徑為 /home/student/ansible/roles
```

:::spoiler Reference
```yaml=
inventory=/home/student/ansible/inventory
collections_path=/home/student/ansible/collections
roles_path=/home/student/ansible/roles
```
:::

### Ansible vault
```
請下載 http://content.example.com/pub/secret.yml 存放在 /home/student/ansible
這個檔案已被ansible‐vault 加密，請將加密密碼由原本的 abc123 改為 soeasy
```
```
建立 Ansible vault 存放使用者密碼:
vault 檔案 /home/student/ansible/vault.yml
pw_developer: devpw
pw_manager: mgrpw
加解密 vault 檔案的密碼為 abc123
密碼存放在 /home/student/ansible/vault-password.txt
```
::: spoiler
```shell=
curl http://content.example.com/pub/secret.yml --output /home/student/ansible/secret.yml
ansible-vault rekey /home/student/ansible/secret.yml
```

```shell=
ansible-vault create home/student/ansible/vault.yml
echo "abc123" > /home/student/ansible/vault-password.txt
```
:::


### 修改檔案內容, 有使用到 magic variable
```
請建立一個 /home/student/ansible/issue.yml 的 playbook 檔案:
執行在所有主機, 任務為修改所有主機的 /etc/issue
替換 /etc/issue 的內容為一行的文字內容:
developer 群組下的主機改為 Development
test 群組下的主機改為 Test
production 群組下的主機改為 Production
```
::: spoiler Reference
```yaml=
# 新增ansible.cfg
# 新增inventory
---
- name: change issue file
  hosts: all
  become: true
  tasks:
    - name: change issue text
      ansible.builtin.copy:
        dest: /etc/issue
        content: "{{ group_names[0] | capitalize }}"
```
:::

### 產生 hardware 報告
有些主機可能沒有vdb、或大小不一致，有使用 magic variable
```
建立一個 playbook 為 /home/student/ansible/hardware_rp.yml, 
執行時可以在所有主機產生一個檔案 /root/hardware_rp.txt, 內容為(key=value):
Inventory host name
Total memory in MB
BIOS version
Size of disk device vda
Size of disk device vdb
```
::: spoiler reference
```yaml=
---
- name: gen hardware report
  hosts: all
  become: true
  tasks:
    - name: fetch inventory
      ansible.builtin.get_url:
        url: http://content.example.com/pub/hardware_rp.test
        dest: /root/hardware_rp.txt
      ignore_errors: true
    - name: copy
      ansible.builtin.copy:
        src: hardware_rp.txt
        dest: /root/hardware_rp.txt
    - name: Inventory hostname
      ansible.builtin.lineinfile:
        path: /root/hardware_rp.txt
        regex: '^HOST='
        line: 'HOST={{ inventory_hostname }}'
    - name: Inventory memory
      ansible.builtin.lineinfile:
        path: /root/hardware_rp.txt
        regex: '^MEMORY='
        line: 'MEMORY={{ ansible_facts["memtotal_mb"] }}'
    - name: Inventory bios
      ansible.builtin.lineinfile:
        path: /root/hardware_rp.txt
        regex: '^BIOS='
        line: 'BIOS={{ ansible_facts["bios_version"] }}'
    - name: test
      ansible.builtin.debug:
        msg: ansible_facts
    - name: Inventory vda size
      ansible.builtin.lineinfile:
        path: /root/hardware_rp.txt
        regex: '^DISK_SIZE_VDA='
        line: 'DISK_SIZE_VDA={{ ansible_facts["devices"]["vda"]["size"] }}'
    - name: Inventory vdb size
      ansible.builtin.lineinfile:
        path: /root/hardware_rp.txt
        regex: '^DISK_SIZE_VDB='
        line: 'DISK_SIZE_VDB={{ ansible_facts["devices"]["vdb"]["size"] }}'
      when: ansible_facts["devices"]["vdb"]["size"] is defined

    - name: Inventory vdb size
      ansible.builtin.lineinfile:
        path: /root/hardware_rp.txt
        regex: '^DISK_SIZE_VDB='
        line: 'DISK_SIZE_VDB=NONE'
      when: ansible_facts["devices"]["vdb"]["size"] is not defined
```
:::

### 產生 hosts file
```
下載 template file http://content.example.com/pub/hosts.j2 存放在 /home/student/ansible
完成 template 用來產生像 /etc/hosts 檔案格式的一個文字檔
建立 playbook 在 /home/student/ansible/hosts.yml, 使用修改過的 template 產生 /etc/hosts.bak 在 developer 群組裡的主機
```
::: spoiler Reference

```yaml=
---
- name: exercise
  hosts: developer
  become: true
  tasks:
    - name: jinja
      ansible.builtin.template:
        src: hosts.j2
        dest: /etc/hosts.bak
```
`cat hosts.j2`
```jinja=
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

{% for i in groups['all'] %}
{{ hostvars[i]['ansible_facts']['default_ipv4']['address'] }} {{ hostvars[i]['ansible_facts']['fqdn'] }}  {{ hostvars[i]['ansible_facts']['hostname'] }}
{% endfor %}
```
:::

### 安裝 Collection
```
安裝 http://content.example.com/pub/ 下的 collection 
	redhat-rhel_system_roles-1.16.2.tar.gz  (fedora.linux_system_roles)
	ansible-posix-1.4.0.tar.gz
	community-general-4.3.0.tar.gz
要安裝在預設的 collections 目錄 /home/student/ansible/collections
```
::: spoiler reference
```bash=
ansible-galaxy collection list
cd /home/student/ansible
mkdir -p collections
echo "collections_paths=./collections" >> ansible.cfg
ansible-galaxy collection install -p collections/ redhat-rhel_system_roles-1.16.2.tar.gz
ansible-galaxy collection install -p collections/ ansible-posix-1.4.0.tar.gz
ansible-galaxy collection install -p collections/ community-general-4.3.0.tar.gz
```
:::


### 建立 dnf 倉儲 
```
Repository 1:
	倉儲名稱 BaseOS
	倉儲描述 BaseOS software
	倉儲 URL 為 http://content.example.com/rhel9.0/x86_64/dvd/BaseOS
	GPG 檢核功能要啟用
	GPG key URL 為 http://content.example.com/pub/RPM-GPG-KEY-redhat-release
	倉儲的狀態要啟用

Repository 2:
	倉儲名稱 AppStream
	倉儲描述 AppStream software
	倉儲 URL 為 http://content.example.com/rhel9.0/x86_64/dvd/AppStream
	GPG 檢核功能要啟用
	GPG key URL 為 http://content.example.com/pub/RPM-GPG-KEY-redhat-release
	倉儲的狀態要啟用 
```
::: spoiler reference
```yaml=
---
- name: repo
  hosts: dev
  become: true
  tasks:
    - name: Add multiple repositories into the same file (1/2)
      ansible.builtin.yum_repository:
        name: BaseOS
        description: BaseOS software
        baseurl: http://content.example.com/rhel9.0/x86_64/dvd/BaseOS
        gpgcheck: yes
        gpgkey: http://content.example.com/pub/RPM-GPG-KEY-redhat-release
        enabled: true

    - name: Add multiple repositories into the same file (2/2)
      ansible.builtin.yum_repository:
        name: AppStream
        description: AppStream software
        baseurl: http://content.example.com/rhel9.0/x86_64/dvd/AppStream
        gpgcheck: yes
        gpgkey: http://content.example.com/pub/RPM-GPG-KEY-redhat-release
        enabled: true
```
```yaml=
# wildcard
# state: latest
---
- name: main
  hosts: production, test, developer
  become: true
  tasks:
    - name: install php mariadb
      ansible.builtin.dnf:
        name:
          - php
          - mariadb
        state: present

- name: main2
  hosts: developer
  become: true
  tasks:
    - name: rpm
      ansible.builtin.dnf:
        name: "@RPM Development Tools"
        state: present
    - name: rpm
      ansible.builtin.dnf:
        name: "*"
        state: latest
```
:::

### system-role
```
Runs on all managed hosts
使用 timesync role
使用目前系統作用中的 NTP provider
NTP server 為 classroom.example.com
開啟iburst 參數 

使用 selinux role
將SELinux policy 設定為 targeted
設定SELinux 的運作模式為 enforcing
```
::: spoiler reference
```yaml=
# 先安裝system-role collections
# 查看readme.md
---
- name: shit
  hosts: all
  become: true
  become_method: sudo
  vars:
    timesync_ntp_provider: chrony # 注意一下版本
    timesync_ntp_servers:
      - hostname: classroom.example.com
        iburst: yes
    selinux_policy: targeted
    selinux_state: enforcing
  roles:
    - redhat.rhel_system_roles.timesync
    - redhat.rhel_system_roles.selinux
```
Check
```bash=
cat /etc/chrony.conf
sestatus
```
:::

### 建立一個 role
```
建立一個 role 在 /home/student/ansible/roles 目錄, 命名為 apache
建立的 role 要能執行以下事項:
httpd 套件的安裝, 系統啟動時自動啟動, 即刻啟動
firewall 規則的設定允許 web server 能被存取
建立 template 檔案 index.html.j2, 用來建立 /var/www/html/index.html, 檔案內容為:

You are at HOSTNAME on IPADDRESS

HOSTNAME 為 managed node 的 FQDN, IPADDRESS 為 managed node 的 IP address

建立 playbook  /home/student/ansible/install-apache.yml 
執行在 webservers 群組裡的主機
使用剛才建立的 role
```

::: spoiler reference
```yaml=
---

- name: install-apache.yml
  hosts: webservers
  become: true
  roles:
    - apache
```
```
project
├── roles
│ ├── apache
│ │   ├── tasks
│ │   ├── templates
├── ansible.cfg
├── inventory
└── playbook.yml
```
tasks
```yaml=
---

- name: install httpd
  ansible.builtin.dnf:
    name: httpd
    state: present

- name: enable httpd
  ansible.builtin.service:
    name: httpd
    state: started

- name: firewall rule
  ansible.posix.firewalld:
    service: http
    immediate: true
    state: enabled
    permanent: true

- name: template
  ansible.builtin.template:
    src: index.html.j2
    dest: /var/www/html/index.html
```

templates:
```yaml=
You are at {{ ansible_facts['fqdn'] }} on {{ ansible_facts['default_ipv4']['address'] }}
```
:::

### 使用 requirements.yml 安裝role
```
/home/student/ansible/roles/requirements.yml,
下載並安裝 roles 到 /home/student/ansible/roles

http://content.example.com/pub/haproxy.tar
命名為 nb

http://content.example.com/pub/phpinfo.tar
命名為 pi
```
::: spoiler reference
```yaml=
---
# file單數 兩條// r+ 路徑

- src: file:///home/student/haproxy.tar
  name: nb

- src: file:///home/student/phpinfo.tar
  name: pi
  
```
```bash=
ansible-galaxy role install -p ./roles -r ./roles/requirements.yml
```
:::

### 使用 Ansible Galaxy 的 role 
> 要看一下設定
::: spoiler reference
```yaml=
---
- name: ensure directory existed
  hosts: webservers
  tasks:
    - name: directory
      ansible.builtin.file:
        path: /var/www/html
        state: directory

- name: gather fact
  hosts: webservers
  become: true
  roles:
    - pi

- name: haproxy
  hosts: balancer
  remote_user: devops
  become: true
  roles:
    - nb
  tasks:
    - name: server1
      ansible.posix.firewalld:
        service: http
        permanent: true
        immediate: true
        state: enabled
```
:::
### 建立 web 站台目錄
```
建立 playbook /home/student/ansible/createwebdir.yml:
執行在 production 群組裡的主機
建立 /webapp 目錄
目錄擁有群組為 webapp
權限設定為: owner=read+write+execute, group=read+write+execute, other=read+execute
特殊權限: set group ID
建立 Symbolically link /var/www/html/webapp 指向 /webapp
建立檔案 /webapp/index.html 內容為: Production Test
瀏覽時 (http://serverc.lab.example.com/webapp/) 要能看到以下的輸出:

Production Test 
```

::: spoiler reference
```yaml=
---

- name: easy create folder
  hosts: production
  become: true
  tasks:
    - name: conf selinux
      community.general.sefcontext:
        target: '/webapp(/.*)?'
        setype: httpd_sys_content_t
        state: present

    - name: create group
      ansible.builtin.group:
        name: webapp
        state: present

    - name: create
      ansible.builtin.file:
        path: /webapp
        state: directory
        group: webapp # owner?
        mode: '2775' # checkuid
        setype: httpd_sys_content_t # important

    - name: symbolic
      ansible.builtin.file:
        src: /webapp
        dest: /var/www/html/webapp
        state: link

    - name: file
      ansible.builtin.copy:
        dest: /webapp/index.html
        content: Production Test
        setype: httpd_sys_content_t # important
```
> selinux開一下比較好
:::

### 建立及使用 LV 
```
建立 playbook  /home/student/ansible/disk1.yml 執行在所有主機:
建立 logical volume:
logical volume 建立在 fox volume group
logical volume 命名為 avatar
logical volume 大小為 800 MiB

在 logical volume 上建立 xfs 檔案系統

如果無法建立指定大小的 logical volume, 要出現錯誤訊息:
Could not create logical volume of that size
而且能夠建立 500 MiB 大小的 logical volume

如果 volume group fox 不存在, 要出現錯誤訊息:
Volume group does not exist
```
::: spoiler reference
```yaml=
---

- name: create lv
  hosts: all
  become: true
  tasks:
    - name: fox is existed
      ansible.builtin.fail:
        msg: Volume group does not exist
      when: ansible_facts["lvm"]["vgs"]["fox"] is not defined
    
    - name: create lv block
      block:
        - name: Creates a logical volume
          community.general.lvol:
            vg: fox
            size: 800
            lv: avartar
          notify: create fs

      rescue:
        - name: not enough
          ansible.builtin.debug:
            msg: Could not create logical volume of that size
           
        - name: 500g
          community.general.lvol:
            vg: fox
            size: 500g
            lv: avartar
          notify: create fs

  handlers:
    - name: create fs
      community.general.filesystem:
        fstype: xfs
        dev: /dev/fox/avartar
```
:::

### 建立及使用 Partition
```
建立 playbook  /home/student/ansible/part1.yml 執行在 balancer 主機:
在 vdc 建立1號 partition:
partition 大小為 1500 MiB
在 partition 上建立 ext4 檔案系統
檔案系統要固定掛載在 /space1

在 vdd 建立1號 partition:
partition 大小為 1500 MiB
在 partition 上建立 ext4 檔案系統
檔案系統要固定掛載在 /space2

如果無法建立指定大小的 partition , 要出現錯誤訊息:
Could not create partition of that size
而且能夠建立 800 MiB 大小的 partition

如果 disk 不存在, 要出現錯誤訊息:
Disk does not exist
```

::: spoiler reference
```yaml=
---
- name: Create partition and Filesyste
  hosts: balancer
  become: true
  tasks:
    - name: no vdc
      ansible.builtin.fail:
        msg: Disk does not exist
      when: ansible_facts['devices']['vdc'] is not defined

    - name: create partition1
      block:
        - name: partition 1500m
          community.general.parted:
            device: /dev/vdc
            number: 1
            state: present
            part_start: 1MiB
            part_end: 1501MiB
          notify: 
            - create fs vdc
            - mount fs vdc
      rescue:
        - name: send  message
          ansible.builtin.debug:
            msg: Could not create partition of that size

        - name: partition 800m
          community.general.parted:
            device: /dev/vdc
            number: 1
            state: present
            part_start: 1MiB
            part_end: 801MiB
          notify: 
            - create fs vdc
            - mount fs vdc

    - name: no vdd
      ansible.builtin.fail:
        msg: Disk does not exist
      when: ansible_facts['devices']['vdd'] is not defined

    - name: create partition1
      block:
        - name: partition 1500m
          community.general.parted:
            device: /dev/vdd
            number: 1
            state: present
            part_start: 1MiB
            part_end: 1501MiB
          notify:
            - create fs vdd
            - mount fs vdd
      rescue:
        - name: send  message
          ansible.builtin.debug:
            msg: Could not create partition of that size

        - name: partition 800m
          community.general.parted:
            device: /dev/vdd
            number: 1
            state: present
            part_start: 1MiB
            part_end: 801MiB
          notify:
            - create fs vdd
            - mount fs vdd

  handlers:
    - name: create fs vdc
      community.general.filesystem:
        fstype: ext4
        dev: /dev/vdc1

    - name: mount fs vdc
      ansible.posix.mount:
        path: /space1
        src: /dev/vdc1
        state: mounted
        fstype: ext4

    - name: create fs vdd
      community.general.filesystem:
        fstype: ext4
        dev: /dev/vdd1

    - name: mount fs vdd
      ansible.posix.mount:
        path: /space2
        src: /dev/vdd1
        state: mounted
        fstype: ext4

```
:::

### 建立使用者帳號
```
使用者的 job description 若是 developer:
建立在 developer 和 test 群組裡的 managed hosts
UID 依照 user_list.yml 檔案裡的內容設定
密碼設定為 pw_developer 變數的值(vault.yml),  設定密碼的有效天數為 99 天
帳號的 secondary group 設定為 developers
密碼要使用 SHA512 hash 格式

使用者的 job description 若是 manager:
建立在 production 群組裡的 managed hosts
UID 依照 user_list.yml 檔案裡的內容設定
密碼設定為 pw_manager 變數的值(vault.yml),  設定密碼的有效天數為 99 天 -> password_expire_max
帳號的 secondary group 設定為 manager -> groups
密碼要使用 SHA512 hash 格式 -> password_hash('sha512')

playbook 要能使用 vault password file /home/student/ansible/vault-password.txt 運作
```
::: spoiler reference
```yaml=
---

- name: create user
  hosts: production, test
  become: true
  vars_files:
    - user_list.yml
    - vault.yml

  tasks:
    - name: add group
      ansible.builtin.group:
        name: developers
        state: present

    - name: create user
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: developers
        uid: "{{ item.uid }}"
        password: "{{ pw_developer | password_hash('sha512') }}"
        state: present
        password_expire_max: 99
      loop: "{{ users }}"
      when: item.job == "developer"


- name: create user
  hosts: production
  become: true
  vars_files:
    - user_list.yml
    - vault.yml

  tasks:
    - name: add group
      ansible.builtin.group:
        name: manager
        state: present

    - name: create user
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: manager
        uid: "{{ item.uid }}"
        password: "{{ pw_manager | password_hash('sha512') }}"
        state: present
        password_expire_max: 99
      loop: "{{ users }}"
      when: item.job == "manager"
```
> 題目有要求要用vault_password_file解鎖
> 那就必須要把密碼echo進去
:::

### cron job
```
使用者 devops 設定了 cron job 每 2 分鐘執行以下指令:
logger "EX294 is so easy."
```
::: spoiler reference
```yaml=
---

- name: cronjob
  hosts: all
  become: true
  tasks:
    - name: file
      ansible.builtin.cron:
        name: cron
        minute: "*/2"
        user: devops
        job: 'logger "EX294 is so easy."'
```
:::