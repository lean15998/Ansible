## Inventory host

```sh
root@quynv:~# vim /etc/ansible/hosts 

[local]
127.0.0.1

[node]
10.0.0.52
````

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











