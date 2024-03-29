# 1. Khái niệm

- Trong Ansible, Role là một cơ chế để tách 1 playbook ra thành nhiều file. Việc này nhằm đơn giản hoá việc viết các playbook phức tạp và có thể tái sử dụng lại nhiều lần

- Role là một bộ khung (framework) để chia nhỏ playbook thành nhiều files khác nhau. Mỗi role là một thành phần độc lập, bao gồm nhiều variables, tasks, files, templates, và modules...

- Một role sẽ có 7 folder với các chức năng khác nhau gồm: vars, templates, handlers, files, meta, tasks và defaults. Mỗi một thư mục cần phải chứa 1 file main.yml. Trong đó thì tasks thường là folder quan trọng nhất, thường dùng để chứa những playbook.
- Trong đó

<ul>
  <ul>
    <li> tasks: chứa danh sách các task chính được thực thi trong role này.
    <li> handlers: chứa các handler, có thể được dùng trong role này hoặc các role khác.
    <li> defaults – chứa các biến được dùng default cho role này
    <li> vars – chứa thông tin các biến dùng trong role, biến trong vars sẽ override biến trong default
    <li> files – chứa các file cần dùng để deploy trong role này, cụ thể như file binary, file cài đặt…
    <li> templates – chứa các file template theo jinja format đuôi *.j2 (có thể là file config, file systemd…).
    <li> meta – định nghĩa 1 số metadata của role này, như là dependencies
      </ul>
  </ul>

# 2. Viết role

#### Tạo role bằng lệnh ansible-galaxy

```sh
root@quynv:/etc/ansible/roles# ansible-galaxy init influxdb
- Role influxdb was created successfully
root@quynv:/etc/ansible/roles# ansible-galaxy init telegraf
- Role telegraf was created successfully
root@quynv:/etc/ansible/roles# ansible-galaxy init grafana
- Role grafana was created successfully
```

- Cấu trúc của một role đã tạo

```sh
root@quynv:/etc/ansible/roles# tree grafana/
grafana/
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

### Viết từng thư mục cụ thể

- Tasks (viết các task chính của role)

VD:
```sh
root@quynv:/etc/ansible/roles# cat grafana/tasks/main.yml 

---
# tasks file for grafana
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
            notify:
                    - start grafana
          - name: open port
            ufw:
                    rule: allow
                    port: 3000
                    proto: tcp           
```

- vars (Viết thông tin các biến dùng trong role)

VD:

```sh
root@quynv:/etc/ansible/roles# cat influxdb/vars/main.yml 

---
# vars file for grafana
databasename: "telegraf"
username: "telegraf"
userpass: "telegraf"
ip_database: "10.0.0.52"
```

- handler (Viết các handler)

```sh
root@quynv:/etc/ansible/roles# cat grafana/handlers/main.yml 

---
# handlers file for grafana
          - name: start grafana
            service:
                    name: 'grafana-server'
                    state: started
```

- templates

```sh
root@quynv:/etc/ansible/roles# cat ìnluxdb/templates/template.j2

[meta]
  dir = "/var/lib/influxdb/meta"

[data]
  dir = "/var/lib/influxdb/data"
  wal-dir = "/var/lib/influxdb/wal"
  series-id-set-cache-size = 100

[http]
enable = true
log_enabled = true
flux-enabled = true
bind-address = ':8086'
```

# 3. Chạy role

- Sau khi viết xong role. Ta tạo file host và playbook để khởi chạy


```sh
root@quynv:/etc/ansible/test# vim hosts 

[telegraf]
ubuntu02 ansible_host=10.0.0.53 ansible_port=22 ansible_user=root
centos01 ansible_host=10.0.0.61 ansible_port=22 ansible_user=root

[influxdb]
ubuntu01 ansible_host=10.0.0.52 ansible_port=22 ansible_user=root

[grafana]
ubuntu01 ansible_host=10.0.0.52 ansible_port=22 ansible_user=root
```

```sh
root@quynv:/etc/ansible/test# cat inluxdb-grafana.yml 
---
- name: Setup Monitoring Services
  hosts: ubuntu01
  roles:
      - influxdb
      - grafana
```
```sh
root@quynv:/etc/ansible# cat test/telegraf.yml 
---
- name: Setup Monitoring Services
  hosts: ['centos01','ubuntu02']
  roles:
      - telegraf
```

```sh
root@quynv:/etc/ansible/test# cat createdb.yml 
---
- name: create db
  hosts: ubuntu01
  roles:
      - create_db
```

- Khởi chạy playbook influxdb-grafana.yml

```sh
root@quynv:/etc/ansible/test# ansible-playbook influxdb-grafana.yml

PLAY [Setup Monitoring Services] *****************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [ubuntu01]

TASK [influxdb : add key] ************************************************************************************************************************************************
changed: [ubuntu01]

TASK [influxdb : add repo] ***********************************************************************************************************************************************
changed: [ubuntu01]

TASK [influxdb : apt update] *********************************************************************************************************************************************
changed: [ubuntu01]

TASK [influxdb : install influxdb] ***************************************************************************************************************************************
changed: [ubuntu01]

TASK [influxdb : allow port] *********************************************************************************************************************************************
changed: [ubuntu01]

TASK [influxdb : remove file configure] **********************************************************************************************************************************
changed: [ubuntu01]

TASK [influxdb : Copy grafana.ini file] **********************************************************************************************************************************
changed: [ubuntu01]

TASK [grafana : add key] *************************************************************************************************************************************************
changed: [ubuntu01]

TASK [grafana : add repo] ************************************************************************************************************************************************
changed: [ubuntu01]

TASK [grafana : apt update] **********************************************************************************************************************************************
changed: [ubuntu01]

TASK [grafana : install grafana] *****************************************************************************************************************************************
changed: [ubuntu01] => (item=grafana)
changed: [ubuntu01] => (item=apache2)

TASK [grafana : open port] ***********************************************************************************************************************************************
changed: [ubuntu01]

RUNNING HANDLER [grafana : start grafana] ********************************************************************************************************************************
changed: [ubuntu01]

PLAY RECAP ***************************************************************************************************************************************************************
ubuntu01                   : ok=14   changed=13   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

- Khởi chạy playbook createdb.yml

```sh
root@quynv:/etc/ansible/test# ansible-playbook createdb.yml --ask-vault-pass
Vault password: 

PLAY [create db] *********************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [ubuntu01]

TASK [create_db : create_db] *********************************************************************************************************************************************
ok: [ubuntu01]

TASK [create_db : create_user_db] ****************************************************************************************************************************************
changed: [ubuntu01]

PLAY RECAP ***************************************************************************************************************************************************************
ubuntu01                   : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```


- Khởi chạy playbook telegraf.yml


```sh
root@quynv:/etc/ansible/test# ansible-playbook telegraf.yml 

PLAY [Setup Monitoring Services] *****************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [centos01]
ok: [ubuntu02]

TASK [telegraf : add repo influx] ****************************************************************************************************************************************
skipping: [ubuntu02]
ok: [centos01]

TASK [telegraf : Install telegraf] ***************************************************************************************************************************************
skipping: [ubuntu02]
ok: [centos01]

TASK [telegraf : add key] ************************************************************************************************************************************************
skipping: [centos01]
changed: [ubuntu02]

TASK [telegraf : add repo] ***********************************************************************************************************************************************
skipping: [centos01]
changed: [ubuntu02]

TASK [telegraf : apt update] *********************************************************************************************************************************************
skipping: [centos01]
changed: [ubuntu02]

TASK [telegraf : install influxdb-client] ********************************************************************************************************************************
skipping: [centos01]
changed: [ubuntu02]

TASK [telegraf : download telegraf] **************************************************************************************************************************************
skipping: [centos01]
changed: [ubuntu02]

TASK [telegraf : install telegraf] ***************************************************************************************************************************************
skipping: [centos01]
changed: [ubuntu02]

TASK [telegraf : configure telegraf] *************************************************************************************************************************************
changed: [ubuntu02]
changed: [centos01]

TASK [telegraf : configure telegraf] *************************************************************************************************************************************
changed: [ubuntu02]
changed: [centos01]

TASK [telegraf : configure telegraf] *************************************************************************************************************************************
changed: [ubuntu02]
changed: [centos01]

TASK [telegraf : configure telegraf] *************************************************************************************************************************************
changed: [ubuntu02]
ok: [centos01]

TASK [telegraf : configure telegraf] *************************************************************************************************************************************
changed: [ubuntu02]
changed: [centos01]

RUNNING HANDLER [telegraf : start telegraf] ******************************************************************************************************************************
ok: [ubuntu02]

PLAY RECAP ***************************************************************************************************************************************************************
centos01                   : ok=8    changed=4    unreachable=0    failed=0    skipped=6    rescued=0    ignored=0   
ubuntu02                   : ok=13   changed=11   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```


### Kết quả

- Đăng nhập dashboard
<img src="https://github.com/lean15998/Ansible/blob/main/image/05.png">

- Import influxdb

<img src="https://github.com/lean15998/Ansible/blob/main/image/06.png">

- Thêm dashboard
<img src="https://github.com/lean15998/Ansible/blob/main/image/07.png">  
