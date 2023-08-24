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
# VAR的那張還沒練
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
    failed_when: "'Password missing' in command_result.stdout"
	
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
