# ตูมตาม CSV

__TEST/BASHION VM__

```
VM Dir
	- Dockerfile
	- Vagrantfile
```
คือ โฟลเดอร์ ที่ใช้เก็บ Dockerfile เเละ Vagrantfile สำหรับใช้สร้าง vm เพื่อทดสอบ จะเปิด port 2222, 8080, 4444

**Command to Setup**
```bash
cd ./test-vm
vagrant up
cd ../bashsion-vm
vagrant up
```

__Ansible-host_prepareration__

คือโฟล์เดอร์ที่ใช้ สำหรับ prepareration vm ให้พร้อมสำหรับ ทดสอบ ตามที่พี่ เบียร์สั่ง เเล้วก็มี เพิ่ม nginx เข้าไป

**Command to Setup**
```bash
cd ./ansible-host_preparation
make playbook
```

__ansible_csv__

คือ โฟลเดอร์ที่ใช้สำหรับ check firewall port 

```
.
├── Makefile
├── ansible.cfg
├── connectivity_results.csv
├── data.json
├── inventory.ini
├── log.txt
├── main.yaml
├── tasks
│   ├── check_icmp.yaml
│   ├── check_tcp.yaml
│   ├── check_udp.yaml
│   ├── connect_client.yaml
│   ├── connect_client_UDP.yaml
│   ├── inventory_setup.yaml
│   ├── main.yaml
│   └── read_csv.yaml
├── test10.csv
└── test9.csv
```

<br/>
<br/>
<br/>

__test{ number }.csv__

คือ ไฟล์ ข้อมูลที่ใช้สำหรับ ทดสอบ firewall port

| # | SRC       | ZONE | SRC_Search_Key | Use Interface | SRC_IP       | DST                  | ZONE | DST_Search_Key      | Use Interface | DST_IP                         | TCP         | UDP         | ICMP  | Objective | Remark |
|---|----------|------|---------------|--------------|-------------|----------------------|------|---------------------|--------------|-------------------------------|------------|------------|------|-----------|--------|
|   | Bashion   | SE   | Bashion_SE   | mgmt_ip      | 127.0.0.1    | All_k8s_worker_nodes | SE   | All_k8s_worker_nodes_SE | svc_ip      | 127.0.0.1                      | 4444       | 8888       | TRUE |  |  |
|   | Bashion   | NE   | Bashion_NE   | mgmt_ip      | 127.0.0.2    | Registery            | NE   | Registery_NE        | svc_ip      | "127.0.0.4, 127.0.0.5"         | "4444, 3244" | "8092, 3213" | FALSE |  |  |
|   | Bashion   | NE   | Bashion_NE   | mgmt_ip      | 127.0.0.1    | Registery            | NE   | Registery_NE        | svc_ip      | "127.0.0.9, 127.0.0.8"         | "4443, 3222" | "8032, 3211" | FALSE |  |  |

<br/>
<br/>
<br/>

__connectivity_results.csv__

คือไฟล์ที่เก็บ result จากการ check firewall port
| SRC         | src_ip     | DST                   | dst_ip     | protocol | port  | success | error_msg                                                              | timestamp            |
|------------|-----------|----------------------|-----------|----------|------|---------|------------------------------------------------------------------------|----------------------|
| Bashion_NE | 127.0.0.1 | Registery_NE        | 127.0.0.8 | TCP      | 3222 | False   | nc: connect to 127.0.0.8 port 3222 (tcp) failed: Connection refused  | 2025-03-10T06:31:25Z |
| Bashion_NE | 127.0.0.1 | Registery_NE        | 127.0.0.8 | UDP      | 8032 | False   |                                                                        | 2025-03-10T06:31:44Z |
| Bashion_SE | 127.0.0.1 | All_k8s_worker_nodes_SE | 127.0.0.1 | ICMP     |      | False   |                                                                        |                      |

<br/>
<br/>
<br/>

__main.yaml__

คือ ไฟล์ หลักที่ใช้ สำหรับ check firewall เเละ มี step ดังนี้

เริ่มจาก 
- read_csv.yaml
- inventory_setup.yaml
- check_tcp.yaml
- check_udp.yaml
- connect_client.yaml
- connect_client_UDP.yaml
- check_icmp.yaml

<br/>
<br/>
<br/>

__read_csv.yaml__

```
เช็ค csv file มีอยุ่จริง ไหม
- name: Check if CSV file exists
  stat:
    path: "{{ csv_file_path }}"
  register: csv_file_stat

- name: Fail if CSV file doesn't exist
  fail:
    msg: "CSV file not found at {{ csv_file_path }}"
  when: not csv_file_stat.stat.exists


อ่าน csv file ด้วย community.general.read_csv
- name: Read CSV file
  community.general.read_csv:
    path: "{{ csv_file_path }}"
    delimiter: ","
    dialect: excel
  register: csv_data
```

```
หลังจาก อ่านไฟล์ csv เเล้วก็นำ มาเเยก dst_ip src_ip TCP UDP ICMP เเละ assign ไปที่ processed_data
- name: Convert IP string and Port string to lists
  set_fact:
    processed_data: "{{ processed_data | default([]) + 
      [item | combine({
        'dst_ip': (item['DST_IP'] | default('')).split(',') | map('trim') | list,
        'src_ip': (item['SRC_IP'] | default('')).split(',') | map('trim') | list,
        'TCP':    (item['TCP'] | default('')).split(',') | map('trim') | list,
        'UDP':    (item['UDP'] | default('')).split(',') | map('trim') | list,
        'ICMP':    (item['ICMP'] | default('')).split(',') | map('trim') | list,
      })] }}"
  loop: "{{ csv_data.list }}"
  when: item['DST_IP'] != '' or item['SRC_IP'] != '' 

- name: debug Processed_data
  debug:
    msg: "{{ processed_data }}"
```

<br/>
<br/>
<br/>

__inventory_setup.yaml__

```
สร้าง array ขึ้นมา สำหรับ เก็บ ip ของ เเต่ละ service คือ TCP UDP ICMP
- name: Initialize protocol-specific arrays
  set_fact:
    TCP: []
    UDP: []
    ICMP: []

จะทำการ nested loop processed_data เเละ เข้าถึง src_up dst_up TCP เเละ เก็บไว้ในตัวเเปล TCP เเละ ทำเเบบนี้กับ UDP ICMP 	 

- name: Generate TCP entries
  set_fact:
    TCP: "{{ TCP + [{'SRC': item.0.SRC_Search_Key, 'DST': item.0.DST_Search_Key, 'src_ip': item.0.src_ip[item.1], 'dst_ip': item.0.dst_ip[item.2], 'TCP_port': item.0.TCP[item.3]}] }}"
  with_nested:
    - "{{ processed_data }}"
    - "{{ range(0, processed_data | map(attribute='src_ip') | map('length') | max) }}"
    - "{{ range(0, processed_data | map(attribute='dst_ip') | map('length') | max) }}"
    - "{{ range(0, processed_data | map(attribute='TCP') | map('length') | max) }}"
  when:
    - item.0.src_ip is defined and item.0.src_ip | length > item.1
    - item.0.dst_ip is defined and item.0.dst_ip | length > item.2
    - item.0.TCP is defined and item.0.TCP | length > item.3
    - item.0.TCP[item.3] | trim != ''

ืทำการ loop เข้าไปใน TCP เเละ add_host เป็น
 name: Add TCP clients to inventory
  add_host:
    name: "{{ item.SRC }}_client_{{ item.src_ip }}_{{ item.dst_ip }}_{{ item.TCP_port }}"
    groups: "{{ item.SRC }}_{{ item.DST }}_TCP"
    ansible_host: "{{ item.src_ip }}"
    ansible_port: 2223
    ansible_user: "vagrant"
    ansible_password: "vagrant"
    dst_ip: "{{ item.dst_ip }}"
    tcp_port : "{{ item.TCP_port }}"
    src_name: "{{ item.SRC }}"
    dst_name: "{{ item.DST }}"
  loop: "{{ TCP }}"
```