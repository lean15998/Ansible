# 1. Tổng quan

- Ansible là một trong những công cụ quản lý cấu hình hiện đại, nó tạo điều kiện thuận lợi cho công việc cài đặt, quản lý và bảo trì các server từ xa, với thiết kế tối giản giúp người dùng cài đặt và chạy nhanh chóng.Người dùng viết các tập lệnh cấp phép Ansible trong YAML, một tiêu chuẩn tuần tự hóa dữ liệu thân thiện với người dùng, chúng không bị ràng buộc với bất kỳ ngôn ngữ lập trình nào.

- Ansible cung cấp độ tin cậy, tính nhất quán và khả năng mở rộng cho cơ sở hạ tầng CNTT của bạn. Bạn có thể tự động hóa cấu hình cơ sở dữ liệu, lưu trữ, mạng, tường lửa bằng Ansible. Nó đảm bảo rằng tất cả các gói cần thiết và tất cả các phần mềm khác đều nhất quán trên máy chủ để chạy ứng dụng.

## Kiến trúc 
- Ansible sử dụng kiến trúc agentless không cần đến agent để giao tiếp với các máy khác. Cơ bản nhất là giao tiếp thông qua các giao thức WinRM trên Windows, SSH trên Linux hoặc giao tiếp qua chính API của thiết bị đó cung cấp.
Ansible có thể giao tiếp với rất nhiều OS, platform và loại thiết bị khác nhau từ Ubuntu, VMware, CentOS, Windows cho tới Azure, AWS, các thiết bị mạng Cisco và Juniper,… mà hoàn toàn không cần agent khi giao tiếp.

- Nhờ vào cách thiết kế này đã giúp làm tăng tính tiện dụng của Ansible do không cần phải cài đặt và bảo trì agent trên nhiều host. Có thể nói rằng đây chính là một thế mạnh của Ansible so với các công cụ có cùng chức năng như Chef, SaltStack, Puppet (trong đó Salt có hỗ trợ cả 2 mode là agent và agentless).

<img src= "https://github.com/lean15998/Ansible/blob/main/image/01.png" >

<ul>
    <ul>
        <li> Ansible Mgmt node: Là node quản trị các file inventory, playbook, conf ...
        <li> Hosts: Là các hosts chịu sự quản lý bở ansible mgnt node 
    </ul>
</ul>
    
## Một số thuật ngữ cơ bản

#### Inventory: Là file chứa thông tin những server cần quản lý. File này thường nằm tại đường dẫn /etc/ansible/hosts.

```sh
root@quynv:~# vim /etc/ansible/hosts 

[node]
node02 ansible_host=10.0.0.53 ansible_port=22 ansible_user=root
node01 ansible_host=10.0.0.51 ansible_port=22 ansible_user=root

````
```sh
root@quynv:~# ansible all --list-hosts
  hosts (2):
    node02
    node01
```
#### Module hay các thư viện module cung cấp cho Ansible các thư viện để điều khiển hoặc quản lý tài nguyên trên local hoặc các remote các server.

VD: sử dụng module ping để ping đến tất cả các host

```sh
root@quynv:~# ansible all -m ping
node01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
node02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
#### Playbook: Là file chứa các task được ghi dưới định dạng YAML. Máy controller sẽ đọc các task này trong Playbook sau đó đẩy các lệnh thực thi tương ứng bằng Python xuống các máy con.

```sh
root@quynv:/etc/ansible/playbook# vim apache.yaml 

---
- hosts: local
  tasks: 
    - name: Ping check host
      ping: ~
    - name: Install Apache2
      apt: name=apache2 update_cache=yes
```sh

---
- hosts: local
  tasks:
    - name: Ping check host
      ping: ~
    - name: Install Apache2
      apt: name=apache2 update_cache=yes

root@quynv:/etc/ansible/playbook# ansible-playbook apache.yaml

PLAY [node] **************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [node01]
ok: [node02]

TASK [Ping check host] ***************************************************************************************************************************************************
ok: [node01]
ok: [node02]

TASK [Install Apache2] ***************************************************************************************************************************************************
ok: [node01]
changed: [node02]

PLAY RECAP ***************************************************************************************************************************************************************
node01                     : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node02                     : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
#### Task: Một block ghi lại những tác vụ cần thực hiện trong playbook và các thông số liên quan.

#### Role: Là một tập playbook đã được định nghĩa để thực thi 1 tác vụ nhất định. Nếu bạn có nhiều server, mỗi server thực hiện những tasks riêng biệt. Và khi này nếu chúng ta viết tất cả vào cùng một file playbook thì khá là khó để quản lý. Do vậy roles sẽ giúp bạn phân chia khu vực với nhiệm vụ riêng biệt.

#### Variables: Được dùng để lưu trữ các giá trị và có thể thay đổi được giá trị đó. Để khai báo biến, người dùng chỉ cần sử dụng thuộc tính vars đã được Ansible cung cấp sẵn.

<img src= "https://github.com/lean15998/Ansible/blob/main/image/01.png" >

#### Conditions: Ansible cho phép người dùng điều hướng lệnh chạy hay giới hạn phạm vi để thực hiện câu lệnh nào đó. Hay nói cách khác, khi thỏa mãn điều kiện thì câu lệnh mới được thực thi. Ngoài ra, Ansible còn cung cấp thuộc tính Register, một thuộc tính giúp nhận câu trả lời từ một câu lệnh. Sau đó ta có thể sử dụng chính kết quả đó để chạy những câu lệnh sau.
- Cấu trúc lệnh cơ bản của ansible




```sh
$ ansible [tên host] -m [tên module] -a [tham số truyền vào module]

/*
-i : inventory host. Load thư viện host
-m : gọi module của ansible
-a : command_argument gửi kèm theo module mà ta đang gọi
-u : user
-vvvv : debug option
*/
```

- 
```sh
root@quynv:~# ansible all -m ping
node01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
root@quynv:~# ansible node -m command -a "uptime"
node01 | CHANGED | rc=0 >>
 10:10:38 up  1:33,  2 users,  load average: 0.00, 0.00, 0.00

```











