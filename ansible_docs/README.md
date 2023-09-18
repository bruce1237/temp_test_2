# myAnsible
[youtue video tutorial](https://www.youtube.com/watch?v=EraC1AuWEF8&list=PLT98CRl2KxKEUHie1m24-wkyHpEsa4Y70&index=9)

## prep
need to copy the ssh public key to each slave server under .ssh/authorized_keys, and manually ssh to each slave the FIRST time, so the ansible server can access to each slave

---
# docker create container
`docker run -d --privileged --name ansible_s_b ansible_s_i`

---
# ansible configure file

## ansible.cfg
```
[default]
inventory = inventory/inventory
private_key_file = ~/.ssh/id_rsa
remote_user = Simone


```


## `invertory` file
    
inventory file is a list of IPs for the ansible to deploy to, you can group, assign variable as well
```
[server_a]
172.17.0.3

[server_b]
172.17.0.4

[server_c]
172.17.0.5
```

## test the inventory
run below command:

` ansible all --key-file /root/.ssh/id_rsa  -i inventory -m ping` 

to test is the ansible master can reach each slave.

`all`: 
    
    means running module to all the slave, you can you the tag name to only run the module in a specified slave, such as `ansible server_b --key-file /root/.ssh/id_rsa  -i inventory -m ping` will only run the `ping` at `server_b` which been defined in the inventory file

`--key-file`: indicate which private key to use 

`-i`: indicate which inventory file to use / where the inventory file is

`-m`: indicate which module need to run, in this command, running ping module

response
```
172.17.0.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
172.17.0.5 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
172.17.0.3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
---
## run ansible command as root

`ansible server_b --key-file /root/.ssh/id_rsa  -i inventory -m apt -a update_cache=true --become --ask-become-pass`

`--become`: become sudo user

`--ask-become-pass`: will ask for sudo password

---
## use ansible command to install package / update / upgrade

`ansible all --key-file /root/.ssh/id_rsa  -i inventory -m apt -a "name=vim state=latest"`

`state=latest`: telling install the latest, if installed, then update

using `upgrade=dist` to upgrade all the apt in the slave

## use ansible command to gather facts about slave
`ansible all -i inventories/inventory -m gather_facts --limit 172.24.0.2`
this command will gather all the data about the target server, 
- `--limit IPAddress` used to filter out only the ip server data we wanted

all the key-value data gathered from above command can be used in `when` condition in the playbook

---
# playbook

## create a playbook name as ***.yml

### playbook for apt update /  install package

```
---
- hosts: all
  become: true
  tasks:

  - name: install  nano package
    apt:
      name: nano
```

or install multi packages

```
---
- hosts: all
  become: true
  tasks:

  - name: install  nano package
    apt:
      name:
        - nano
        - netcat
```                     
or update repository index
```
- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes
```

### playbook for remove package
normally `state: absent` will do the remove, 
as the netcat need to use `apt autoremove`, so in 
ansible used autoremove and purge

`state` used to describe the state of the package, 
values:
- absent: to remove the package
- build-dep: ensures the package build dependencies are installed
- fixed: attempt to correct a system with broken dependencies in place.
- latest: every time the play book ran, will install the latest version of the package



```
- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes
    apt:
      name:
        - netcat
      autoremove: yes
      purge: yes
      state: absent
```      

## command to run playbook
`ansible-playbook -i inventory ***.yml` 

## clear the ssh hosts
`ssh-keygen -f "/root/.ssh/known_hosts" -R "172.24.0.3"`

# when condition

`when`: only run the play when condition is true, can using: `and`, `or` ... operators

```
- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes
    when: ansible_distribution == "Ubuntu"

    apt:
      name:
        - netcat
      state: latest
    when: ansible_distribution in ["Ubuntu", "Centos", "Debian"]
```    
# clear up playbook

## move `update_cache` into apt 

```
- hosts: all
  become: true
  tasks:

    apt:
      name:
        - tree
        - neofetch
      state: latest
      update_cache: yes
    when: ansible_distribution in ["Ubuntu", "Centos", "Debian"]
``` 

# using variable in play
first define the variable in the inventory file then use it in the play

## using `package` instead of OS-related package manager

> inventory 
```
172.24.0.3 neofetch_pk=neofetch tree_pk=tree
172.24.0.2 neofetch_pk=neofetch tree_pk=tree
172.24.0.6 neofetch_pk=neofetch tree_pk=tree
```

> playbook 
```
- hosts: all
  become: true
  tasks:

  - name: update repository index
    package:
      update_cache: yes
      name:
        - "{{ neofetch_pk }}"
        - "{{ tree_pk }}"
      state: latest
    when: ansible_distribution == "Ubuntu"
```
# targeting specific nodes (grouped servers)
for doing that group the servers in inventory
> grouped_inventory
```
[group_a]
172.24.0.6

[group_b]
172.24.0.4

[group_c]
172.24.0.5

```

then change the hosts to the hosts group name defined in the inventory file
> group_install_playbook.yml
```
- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes

- hosts: group_a
  become: true
  tasks:
    
  - name: install tree
    package:
      name:
        - tree
        - cmatrix
      state: latest

- hosts: group_b
  become: true
  tasks:
  - name: install neofetch
    package:
      name:
        - neofetch
        - cmatrix
      state: latest

- hosts: group_c
  become: true
  tasks:
  - name: install sl
    package: 
      name: sl
      state: latest
```

# tag
tags used as a filter, defined in the playbook, 

- `tags: always`: special tag: `always` will always run regardless tag filter

- fetch hosts from the playbook

  `ansible-playbook -i inventory playbook.yml --list-hosts`
- fetch available tags from the playbook

  `ansible-playbook -i inventory playbook.yml --list-tags` 

to run the playbook by tags

`ansible-playbook -i inventory playbook.yml --tags=tagnameA,tagnameB`

or 

`ansible-playbook -i inventory --tags tagnameA playbook.yml` 

or for multiple tags

`ansible-playbook -i inventory --tags "tagnameA, tagnameB" playbook.yml` 

# copy file from git/repo to the slave

try file from `https://github.com/bruce1237/smarsh/archive/refs/heads/main.zip`

> still don't know how to copy file in git private repo

```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes

- hosts: group_a
  become: true
  tasks:
    
  - name: install unzip
    package:
      name: unzip
      state: latest
  
  - name: copy file from git to server(group_a)
    unarchive:
      src: https://github.com/bruce1237/smarsh/archive/refs/heads/main.zip
      dest: /var/www/php
      remote_src: yes
      owner: root
      group: root
      mode: 0755

  - name: copy file from local
    copy:
      src: ../files/testCopy.txt #the src is based on where the playbook is, not where the ansible-playbook been ran
      dest: /var/www/test/copyTest.a
      owner: root
      group: root
      mode: 0644 
```

## managing service

> for `register` and `variable.changed`

> beware of the variable can not be used in the following way,
>
> play1 register: variableA
>
> play2 register: variableA
>
> play3 when variableA.changed
>
> if the paly1 run, then the variableA changed, 
>
> play2 ran, didn't change, the variableA stay unchanged,
>
> when come to play3, the paly3 will NOT run as the variableA is not changed.ßß

```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes

- hosts: group_a
  become: true
  tasks:
    
  - name: install httpd
    tags: centos, httpd
    package:
      name: httpd
      state: latest
    when: ansible_distribution == "CentOS"

  
  - name: start httpd service(CentOS)
    service:
      name: httpd
      state: started # = service httpd start
      enabled: yes #start httpd service at start up
    when: ansible_distribution == "CentOS"

  - name: change e-mail address for admin, this requires service reboot
    tags: change_email
    lineinfile: # this lineinfile module allows to change a line in a file
      path: /etc/httpd/conf/httpd.conf
      regexp: '^ServerAdmin'
      line: ServerAdmin somebody@somewhere.net #rewrite the line according to the regexp
    when: ansible_distribution == "CentOS"
    register: variableOfChangeStatus #this just a variable name, 

  - name: reboot httpd service
    tags: apache
    service:
      name: httpd
      state: restarted
    when: variableOfChangeStatus.changed #check if the variable changed, 
    
```    

## add user
this play book create a new user and set to sudo user with ssh-key to access to other server

```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes

- hosts: group_a
  become: true
  tasks:
    
  - name: create user "simone"
    tags: always
    user:
      name: simone
      groups: root

  - name: add ssh key for simone
    tags: always
    authorized_key:
      user: simone
      key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDL0btH60yKhv8AvNHbUgCJUeBIvAS+lpPbGeh7lCSZ5P7iEwYYFeonIQjkwVH4eJ5cROR3R1yx+OEID+QZRxafXp5gLnO4+kRwe3IcaT/EltZF0/Cd/2J8BxaliSus9CNKBAk7D5n2TihoCB4UfkstCOM5tephwrfcwkklzTdN4n3sb/PCJvixAqGviVYd0wd6O96aPg76Rb+xjJ+WUSG16RqYBStJnxI9oRSpUthxbRgwlRuu3BTWPXjv+7zF8ntTCXTrG1EkRE0iOXNtHwg9CSYZuEJsU7vJBE62MsE9ndf2FY7ouSkGAtNZhmU129LIc4i2is2Dw+EAdlT4sfXTcC8EniYDb5ztzwzz4YKzBRApFQVWd5VKZ8ACpaUbIPZLwlKY/Z3nkMXzX7KBMdV3sNsRzGbuhHiO3qWVMsZF+dMouvxxCPuY5L6Ws3OSWvbxcDxCra3uZeSAETj1xqmUPe+DdetEHrsQTfOyUhzOGK8R0bmF7Sado8Q7yFPFhJ0= root@711c409bfa3f"

  - name: add sudoers file for simone
    tags: always
    copy:
      src: ../files/sudoer_simone
      dest: /etc/sudoers.d/simone
      owner: root
      group: root
      mode: 0440
```

## using bootstrapping playbook
the purpose of bootstrap playbook is like a pre-install playbook, to configure the target server to a required condition for rest of the playbook to run.

in the following scripts, the bootstrap.yml need to run to pre-set the user Simone,
then the adduser playbook can run

> bootstrap.yml
```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes

- hosts: group_a
  become: true
  tasks:
    
  - name: create user "simone"
    tags: always
    user:
      name: simone
      groups: root

  - name: add ssh key for simone
    tags: always
    authorized_key:
      user: simone
      key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDL0btH60yKhv8AvNHbUgCJUeBIvAS+lpPbGeh7lCSZ5P7iEwYYFeonIQjkwVH4eJ5cROR3R1yx+OEID+QZRxafXp5gLnO4+kRwe3IcaT/EltZF0/Cd/2J8BxaliSus9CNKBAk7D5n2TihoCB4UfkstCOM5tephwrfcwkklzTdN4n3sb/PCJvixAqGviVYd0wd6O96aPg76Rb+xjJ+WUSG16RqYBStJnxI9oRSpUthxbRgwlRuu3BTWPXjv+7zF8ntTCXTrG1EkRE0iOXNtHwg9CSYZuEJsU7vJBE62MsE9ndf2FY7ouSkGAtNZhmU129LIc4i2is2Dw+EAdlT4sfXTcC8EniYDb5ztzwzz4YKzBRApFQVWd5VKZ8ACpaUbIPZLwlKY/Z3nkMXzX7KBMdV3sNsRzGbuhHiO3qWVMsZF+dMouvxxCPuY5L6Ws3OSWvbxcDxCra3uZeSAETj1xqmUPe+DdetEHrsQTfOyUhzOGK8R0bmF7Sado8Q7yFPFhJ0= root@711c409bfa3f"

  - name: add sudoers file for simone
    tags: always
    copy:
      src: ../files/sudoer_simone
      dest: /etc/sudoers.d/simone
      owner: root
      group: root
      mode: 0440
```

> addUserBootstrap_playbook.yml
```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes
    changed_when: false # not a change no matter what

- hosts: group_a
  become: true
  tasks:
    
  # - name: create user "simone"
  #   tags: always
  #   user:
  #     name: simone
  #     groups: root

  - name: add ssh key for simone
    tags: always
    authorized_key:
      user: simone
      key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDL0btH60yKhv8AvNHbUgCJUeBIvAS+lpPbGeh7lCSZ5P7iEwYYFeonIQjkwVH4eJ5cROR3R1yx+OEID+QZRxafXp5gLnO4+kRwe3IcaT/EltZF0/Cd/2J8BxaliSus9CNKBAk7D5n2TihoCB4UfkstCOM5tephwrfcwkklzTdN4n3sb/PCJvixAqGviVYd0wd6O96aPg76Rb+xjJ+WUSG16RqYBStJnxI9oRSpUthxbRgwlRuu3BTWPXjv+7zF8ntTCXTrG1EkRE0iOXNtHwg9CSYZuEJsU7vJBE62MsE9ndf2FY7ouSkGAtNZhmU129LIc4i2is2Dw+EAdlT4sfXTcC8EniYDb5ztzwzz4YKzBRApFQVWd5VKZ8ACpaUbIPZLwlKY/Z3nkMXzX7KBMdV3sNsRzGbuhHiO3qWVMsZF+dMouvxxCPuY5L6Ws3OSWvbxcDxCra3uZeSAETj1xqmUPe+DdetEHrsQTfOyUhzOGK8R0bmF7Sado8Q7yFPFhJ0= root@711c409bfa3f"

  # - name: add sudoers file for simone
  #   tags: always
  #   copy:
  #     src: ../files/sudoer_simone
  #     dest: /etc/sudoers.d/simone
  #     owner: root
  #     group: root
  #     mode: 0440
```

## roles

> it looks like ansible-galaxy to me

for using roles, need to create the following directory structure

*pre-defined name, can't change
- roles *
  - roleName
    - tasks *
      - main.yml *

> playbook/roles/groupA/tasks/main.yml
```
- name: copy a file to groupA
  copy:
   src: testFile.txt
   dest: /var/file_in_test.txt
   owner: root
   group: root
   mode: 0755
```   
> playbook/roles/groupB/tasks/main.yml
```
- name: install tree to groupB
  package:
   name: tree
   state: latest
```
...


> role-main-playbook.yml
this is the main playbook, which will run through each roles
```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes
    changed_when: false

- hosts: all
  become: true
  roles:
    - base

- hosts: group_a
  become: true
  roles:
    - groupA

- hosts: group_b
  become: true
  roles:
    - groupB

- hosts: group_c
  become: true
  roles:
    - groupC
```

## host variables (host_vars) 

host variables is a file normally named by the hostName.yml or hostIP.yml, 
content of the file is a list of key:value variable

this need to use roles as well.

### how host variables works?
1. create a `host_vars` folder, inside the folder, create each host.yml file
   - normally named by the hostName.yml or hostIP.yml
2. in the roles/roleName/tasks/main.yml, using the variables created in the host.yml
3. in the playbook, to invoke the role 


> 172.24.0.2.yml
```
tree_package: tree
neofetch_package: neofetch
train_package: sl
```
> hostvar_playbook.yml
```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes
    changed_when: false

- hosts: all
  become: true
  roles:
    - base

- hosts: group_c
  become: true
  roles:
    - groupC
```

> roles/groupC/tasks/main.yml
```
- name: install sl in groupC
  package: 
   name: 
    - "{{ train_package }}"
    - "{{ tree_package }}"
    - "{{ neofetch_package }}"
   state: latest
```



## Handlers

as the defect of change of when multiple change(see above), then using `notify` instead of register.

once the notify handler has been invoked, will looking for the current role,
handlers folder.

in the playbook, when notify: whatEverNameIs been invoked, 
ansible will look in the current play/role/handlers/main.yml file
looking for WhatEverNameIs play to execute


>handler_playbook.yml
```
  - name: change e-mail address for admin, this requires service reboot
    tags: change_email
    lineinfile: # this lineinfile module allows to change a line in a file
      path: /etc/httpd/conf/httpd.conf
      regexp: '^ServerAdmin'
      line: ServerAdmin somebody@somewhere.net #rewrite the line according to the regexp
    when: ansible_distribution == "CentOS"
    notify: restart_my_apache_service
```


>roles/roleName/handlers/main.yml
```
- name: restart_my_apache_service
  service:
   name: "{{ apache_service }}"
   state: restarted
```   

or in the roles playbook

> handlerrole_playbook.yml invoke groupA
```
---
- hosts: all
  become: true
  pre_tasks:
  
  - name: update repo
    package:
      update_cache: yes
    changed_when: false

- hosts: all
  become: true
  roles:
    - base

- hosts: group_a
  become: true
  roles:
    - groupA
```
> roles/groupA/tasks/main.yml notify handler: file_has_been_changed
```
- name: copy a file to groupA
  copy:
   src: "{{ file_src }}"
   dest: "{{ file_dest }}"
   owner: "{{ file_owner }}"
   group: "{{ file_group }}"
   mode: "{{ file_mode }}"
  notify: file_has_been_changed
```
can also using multi-handlers
```
  notify:
    - handlerA
    - handlerB
    ...
```

can also using the handlers/main.yml as index file

> roles/testrole/tasks/main.yml
```
##tasks for test role

---
- name: True task
  command: /bin/true
  notify:
  - handler1
  - handler2
```

> roles/testrole/handlers/main.yml
```
## test role handlers
---
  - name: handler1
    import_tasks: "handler1_tasks.yml"

  - name: handler2
    debug: 
      msg: "handler2 debug running"
```
>roles/testrole/handlers/handler1_tasks.yml
```
## handler1_tasks.yml
---
- name: handler1_tasks
  debug: 
    msg: "handler1 tasks_debug running"
```
> roles/testrole/handlers/handler2_tasks.yml
```
## handler1_tasks.yml
---
- name: handler2_tasks
  debug: 
    msg: "handler2 tasks_debug running"
```



---
> roles/groupA/handles/main.yml execute the play if invoked
```
- name: file_has_been_changed
  package:
   name: "{{ train_package }}"
   state: latest
```

> playbooks/host_vars/groupAID.yml define the train_package
```
file_src: testFile.txt
file_dest: /var/www/abc.txt
file_owner: root
file_group: root
file_mode: 0755
train_package: neofetch
```

## Templates
### create a folder under roles/roleName/templates

e.g. sshd_config_ubuntu.j2
```

# This is the sshd server system-wide configuration file.  See
# sshd_config(5) for more information.
...

#	ForceCommand cvs server

# put some variables
AllowUsers {{ ssh_users }}
```

### put the variable value into each files in host_vars 
which defines templateVariable and value as src
```
file_src: testFile.txt
file_dest: /var/www/abc.txt
file_owner: root
file_group: root
file_mode: 0755
train_package: neofetch

# for the j2 templates
ssh_users: "root simone"
ssh_template_file: sshd_config_ubuntu.j2
```

### add a play to use the template in roles/roleName/tasks/main.yml

```
- name: file_has_been_changed
  package:
   name: "{{ train_package }}"
   state: latest

- name: generate sshd_config file from template
  tags: ssh
  template:
    src: "{{ ssh_template_file }}"
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: 0644
  notify: restart_sshd
```

### create handler file to handle notify
> roles/roleName/handlers/main.yml
```
- name: restart_my_sshd
  service:
   name: sshd
   state: restarted
```   

# Ansible PULL

## what is ansible-pull?
- comes with ansible - if you already have ansible installed, you have ansible-pull
- allows servers/workstation to "pull" Ansible playbooks from a Git server, and run locally
- no need to maintain a server, no single point of failure
- leverages the functionality of git to extend ansible
> ansible-pull seems like go to each slave server pull the playbook from git repo and run the paly by the command `ansible-pull`, and `local.yml` is the root of play book by default and has to be in root


1. first created a `local.yml` file in the ansible-pull then push the git repo

    ```
    ---
    - hosts: localhost
      connection: local
      become: true

      tasks:
      - name: install htop
        package:
          name: htop
    ```

2. install ansible in each of the slaves server
3. run the following command in each slave server:
  `sudo ansible-pull -U git@github.com:bruce1237/ansible-pull.git`
    > make sure each slave has access right to the git repo

## change the file structure
```
|-local.yml
|-/tasks
|--/files
```

> local.yml
```
----
- hosts: localhost
  connection: local
  become: true

  pre_tasks:
    - name: update repositories
      package:
        update_cache: yes
      changed_when: False
  
  tasks:
    - include: tasks/packages.yml
```

>packages.yml
```
---

- name: install package
  package: 
    name: 
      - mc
      - htop
```          

### set up all slave servers checkout/pull from git repo automatically
1. create a new file called /tasks/users.yml

    create a sudo user to run ansible jobs in the background, 
    > users.yml
    ```
    ---
    - name: create ansible user
      user:
      name: ansible
      system: yes

    - name: copy sudoers_ansible, set up user(ansible) as sudo
      copy:
        src: files/sudoers_ansible
        dest: /etc/sudoers.d/ansible
        owner: root
        group: root
        mode: 0440
    ```

    >/tasks/files/sudoers_ansible
    ```
    ansible ALL=(ALL) NOPASSWD: ALL
    ```

2. create a cron play: /tasks/cron.yml

    ```
    - name: install cron job(ansible-pull)
      cron:
        user: ansible
        name: "ansible provision" #cron task name
        minute: "*/10" #run every 10 minutes
        job: "/usr/bin/ansible-pull -o -U git@github.com:bruce1237/ansible-pull.git" 
    ```
    `-o`: mean check if there is any change in the repo, if yes, do the job, if no change, stay put

3. add the task to `local.yml`
> local.yml
```
---

- hosts: localhost
  connection: local
  become: true

  pre_tasks:
    - name: update repositories
      package:
        update_cache: yes
      changed_when: False
  
  tasks:
    - include: tasks/users.yml
    - include: tasks/cron.yml
    - include: tasks/packages.yml
```
