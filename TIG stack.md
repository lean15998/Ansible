# Mô hình triển khai

```sh
                                         +-----------+-----------+
                                         |        [Ansible]      |
                                         |                       |
                                         |         Ansible       | 
                                         |           ssh         |
                                         |                       | 
                                         |                       |
                                         +-----------+-----------+    
                                                     |
                                                     |
                                           10.0.0.51 | eth0
                        +----------------------------+----------------------------+
                        |                            |                            |                             
                        |10.0.0.52                   |10.0.0.53                   |10.0.0.61                   
            +-----------+-----------+    +-----------+-----------+    +-----------+-----------+    
            |       [ubuntu01]      |    |       [ubuntu02]      |    |       [centos01]      |     
            |                       |    |                       |    |                       |   
            |                       |    |                       |    |                       |  
            |          ssh          |    |          ssh          |    |          ssh          |    
            |                       |    |                       |    |                       |     
            |                       |    |                       |    |                       |    
            +-----------+-----------+    +-----------------------+    +-----------------------+     

```

### Khai báo file inventory

```sh
root@quynv:~# vim /etc/ansible/hosts

[telegraf]
ubuntu02 ansible_host=10.0.0.53 ansible_port=22 ansible_user=root
centos01 ansible_host=10.0.0.61 ansible_port=22 ansible_user=root

[influxdb]
ubuntu01 ansible_host=10.0.0.52 ansible_port=22 ansible_user=root

[grafana]
ubuntu01 ansible_host=10.0.0.52 ansible_port=22 ansible_user=root
```

### Viết playbook

- influxdb

```sh
---
- hosts: "{{hostname}}"
  tasks:
          - name: add key
            apt_key:
                    url: "https://repos.influxdata.com/influxdb.key"
                    state: present
            when: ansible_distribution == "Ubuntu"
          - name: add repo
            ansible.builtin.apt_repository:
                    repo: deb https://repos.influxdata.com/ubuntu focal stable
                    state: present
            when: ansible_distribution == "Ubuntu"
          - name: apt update
            apt:
                    update_cache: yes
            when: ansible_distribution == "Ubuntu"
          - name: install influxdb
            apt:
                    name: ['influxdb','python3-influxdb']
                    state: latest
            when: ansible_distribution == "Ubuntu"
          - name: start influxdb
            service:
                    name: 'influxdb'
                    state: started
          - name: allow port
            ufw:
                    rule: allow
                    port: 8086
                    proto: tcp
            when: ansible_distribution == "Ubuntu"
          - name: configure influxdb
            ini_file:
                    path: /etc/influxdb/influxdb.conf
                    section: http
                    option: "{{item}}"
                    value: 'true'
            with_items:
                    - enable
                    - log_enabled
                    - flux-enabled

          - name: configure influxdb port
            ini_file:
                    path: /etc/influxdb/influxdb.conf
                    section: http
                    option: "bind-address"
                    value: ':8086'
          - name: create_db
            influxdb_database:
                    hostname: 127.0.0.1
                    database_name: "{{databasename}}"
                    state: 'present'
          - name: create_user_db
            influxdb_user:
                    hostname: 127.0.0.1
                    user_name: "{{username}}"
                    user_password: "{{userpass}}"
                    state: 'present'
                    grants:
                            - database: "{{databasename}}"
                              privilege: 'all'

```

- telegraf

```sh
---
- hosts: "{{hostname}}"
  tasks:
          - name: add repo influx
            yum_repository:
                    description: Influxdb upstream yum repo
                    name: influxdb
                    baseurl: "https://repos.influxdata.com/rhel/$releasever/$basearch/stable"
                    gpgcheck: yes
                    enabled: yes
                    gpgkey: "https://repos.influxdata.com/influxdb.key"
            when: ansible_distribution == "CentOS"
          - name: Install telegraf
            yum:
                    name: 'telegraf'
                    state: latest
            when: ansible_distribution == "CentOS"
          - name: add key
            apt_key:
                    url: "https://repos.influxdata.com/influxdb.key"
                    state: present
            when: ansible_distribution == "Ubuntu"
          - name: add repo
            ansible.builtin.apt_repository:
                    repo: deb https://repos.influxdata.com/ubuntu focal stable
                    state: present
            when: ansible_distribution == "Ubuntu"
          - name: apt update
            apt:
                    update_cache: yes
            when: ansible_distribution == "Ubuntu"
          - name: install influxdb-client
            apt:
                    name: 'influxdb-client'
                    state: latest
            when: ansible_distribution == "Ubuntu"
          - name: download telegraf
            get_url:
                    url: https://dl.influxdata.com/telegraf/releases/telegraf_1.21.4-1_amd64.deb
                    dest: /tmp/telegraf.deb
            when: ansible_distribution == "Ubuntu"
          - name: install telegraf
            apt:
                    deb: /tmp/telegraf.deb
            when: ansible_distribution == "Ubuntu"
          - name: start telegraf
            service:
                    name: 'telegraf'
                    state: started
          - name: configure telegraf
            ini_file:
                    path: /etc/telegraf/telegraf.conf
                    section: '[outputs.influxdb]'
                    option: "database"
                    value: '"{{database}}"'
          - name: configure telegraf
            ini_file:
                    path: /etc/telegraf/telegraf.conf
                    section: '[outputs.influxdb]'
                    option: "username"
                    value: '"{{username}}"'
          - name: configure telegraf
            ini_file:
                    path: /etc/telegraf/telegraf.conf
                    section: '[outputs.influxdb]'
                    option: "password"
                    value: '"{{password}}"'
          - name: configure telegraf
            ini_file:
                    path: /etc/telegraf/telegraf.conf
                    section: '[outputs.influxdb]'
                    option: "urls"
                    value: "['http://{{ip_database}}:8086']"
          - name: configure telegraf
            ini_file:
                    path: /etc/telegraf/telegraf.conf
                    section: 'agent'
                    option: "hostname"
                    value: '"{{hostname}}"'
          - name: restart telegraf
            service:
                    name: 'telegraf'
                    state: restarted

```


- Grafana

```sh
---
- hosts: "{{hostname}}"
  tasks:
          - name: add key
            apt_key:
                    url: "https://packages.grafana.com/gpg.key"
                    state: present
            when: ansible_distribution == "Ubuntu"
          - name: add repo
            ansible.builtin.apt_repository:
                    repo: deb https://packages.grafana.com/oss/deb stable main
                    state: present
            when: ansible_distribution == "Ubuntu"
          - name: apt update
            apt:
                    update_cache: yes
          - name: install grafana
            apt:
                    name: '{{item}}'
                    state: latest
            with_items:
                    - grafana
                    - apache2
          - name: start grafana
            service:
                    name: 'grafana-server'
                    state: started
          - name: open port
            ufw:
                    rule: allow
                    port: 3000
                    proto: tcp           

  ```
  
### Chạy playbook 

- Triển khai influxdb trên ubuntu01
  
```sh
root@quynv:/etc/ansible/playbook# ansible-playbook influxdb.yaml --extra-vars '{"hostname":"influxdb","databasename":"telegraf","username":"telegraf","userpass":"telegraf"}'

PLAY [influxdb] **********************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [ubuntu01]

TASK [add key] ***********************************************************************************************************************************************************
changed: [ubuntu01]

TASK [add repo] **********************************************************************************************************************************************************
changed: [ubuntu01]

TASK [apt update] ********************************************************************************************************************************************************
changed: [ubuntu01]

TASK [install influxdb] **************************************************************************************************************************************************
changed: [ubuntu01]

TASK [start influxdb] ****************************************************************************************************************************************************
changed: [ubuntu01]

TASK [allow port] ********************************************************************************************************************************************************
changed: [ubuntu01]

TASK [configure influxdb] ************************************************************************************************************************************************
changed: [ubuntu01] => (item=enable)
changed: [ubuntu01] => (item=log_enabled)
changed: [ubuntu01] => (item=flux-enabled)

TASK [configure influxdb port] *******************************************************************************************************************************************
changed: [ubuntu01]

TASK [create_db] *********************************************************************************************************************************************************
ok: [ubuntu01]

TASK [create_user_db] ****************************************************************************************************************************************************
changed: [ubuntu01]

PLAY RECAP ***************************************************************************************************************************************************************
ubuntu01                   : ok=11   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

-  Triển khai telegraf trên ubuntu02 và centos01

```sh
root@quynv:/etc/ansible/playbook# ansible-playbook telegraf1.yaml --extra-vars '{"hostname":"centos01","database":"telegraf","username":"telegraf","password":"telegraf","ip_database":"10.0.0.52"}'

PLAY [centos01] **********************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [centos01]

TASK [add repo influx] ***************************************************************************************************************************************************
ok: [centos01]

TASK [Install telegraf] **************************************************************************************************************************************************
ok: [centos01]

TASK [add key] ***********************************************************************************************************************************************************
skipping: [centos01]

TASK [add repo] **********************************************************************************************************************************************************
skipping: [centos01]

TASK [apt update] ********************************************************************************************************************************************************
skipping: [centos01]

TASK [install influxdb-client] *******************************************************************************************************************************************
skipping: [centos01]

TASK [download telegraf] *************************************************************************************************************************************************
skipping: [centos01]

TASK [install telegraf] **************************************************************************************************************************************************
skipping: [centos01]

TASK [start telegraf] ****************************************************************************************************************************************************
ok: [centos01]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [centos01]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [centos01]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [centos01]

TASK [configure telegraf] ************************************************************************************************************************************************
ok: [centos01]

TASK [configure telegraf] ************************************************************************************************************************************************
ok: [centos01]

TASK [restart telegraf] **************************************************************************************************************************************************
changed: [centos01]

PLAY RECAP ***************************************************************************************************************************************************************
centos01                   : ok=10   changed=4    unreachable=0    failed=0    skipped=6    rescued=0    ignored=0   
```
```sh
root@quynv:/etc/ansible/playbook# ansible-playbook telegraf1.yaml --extra-vars '{"hostname":"ubuntu02","database":"telegraf","username":"telegraf","password":"telegraf","ip_database":"10.0.0.52"}'

PLAY [ubuntu02] **********************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [ubuntu02]

TASK [add repo influx] ***************************************************************************************************************************************************
skipping: [ubuntu02]

TASK [Install telegraf] **************************************************************************************************************************************************
skipping: [ubuntu02]

TASK [add key] ***********************************************************************************************************************************************************
changed: [ubuntu02]

TASK [add repo] **********************************************************************************************************************************************************
changed: [ubuntu02]

TASK [apt update] ********************************************************************************************************************************************************
changed: [ubuntu02]

TASK [install influxdb-client] *******************************************************************************************************************************************
changed: [ubuntu02]

TASK [download telegraf] *************************************************************************************************************************************************
changed: [ubuntu02]

TASK [install telegraf] **************************************************************************************************************************************************
changed: [ubuntu02]

TASK [start telegraf] ****************************************************************************************************************************************************
ok: [ubuntu02]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [ubuntu02]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [ubuntu02]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [ubuntu02]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [ubuntu02]

TASK [configure telegraf] ************************************************************************************************************************************************
changed: [ubuntu02]

TASK [restart telegraf] **************************************************************************************************************************************************
changed: [ubuntu02]

PLAY RECAP ***************************************************************************************************************************************************************
ubuntu02                   : ok=14   changed=12   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```
- Triển khai grafana trên ubuntu01

```sh
root@quynv:/etc/ansible/playbook# ansible-playbook grafana.yaml --extra-vars '{"hostname":"grafana"}'

PLAY [grafana] ***********************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [ubuntu01]

TASK [add key] ***********************************************************************************************************************************************************
changed: [ubuntu01]

TASK [add repo] **********************************************************************************************************************************************************
changed: [ubuntu01]

TASK [apt update] ********************************************************************************************************************************************************
changed: [ubuntu01]

TASK [install grafana] ***************************************************************************************************************************************************
changed: [ubuntu01] => (item=grafana)
changed: [ubuntu01] => (item=apache2)

TASK [start grafana] *****************************************************************************************************************************************************
changed: [ubuntu01]

TASK [open port] *********************************************************************************************************************************************************
changed: [ubuntu01]

PLAY RECAP ***************************************************************************************************************************************************************
ubuntu01                   : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```


### Kết quả

<img src="https://github.com/lean15998/Ansible/blob/main/image/05.png">


<img src="https://github.com/lean15998/Ansible/blob/main/image/06.png">

<img src="https://github.com/lean15998/Ansible/blob/main/image/07.png">  
