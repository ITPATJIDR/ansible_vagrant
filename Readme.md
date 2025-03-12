# พี่ ตูมตาม CSV

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

__Terraform VM__
```
Terraform Dir
  id_rsa
  id_rsa.pub
  main.tf
```
คือ โฟลเดอร์ที่ใช้เก็บ terraform file หลังจาก apply เสร็จเเล้ว ก็เอา ip ที่ได้จาก output มาเปลียนใน ansible-prepare inventory

**Command to Setup**
```bash
cd ./terraform
make apply
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

เช็ค csv file มีอยุ่จริง ไหม
```
- name: Check if CSV file exists
  stat:
    path: "{{ csv_file_path }}"
  register: csv_file_stat
- name: Fail if CSV file doesn't exist
  fail:
    msg: "CSV file not found at {{ csv_file_path }}"
  when: not csv_file_stat.stat.exists

```
อ่าน csv file ด้วย community.general.read_csv
```
- name: Read CSV file
  community.general.read_csv:
    path: "{{ csv_file_path }}"
    delimiter: ","
    dialect: excel
  register: csv_data
```

หลังจาก อ่านไฟล์ csv เเล้วก็นำ มาเเยก dst_ip src_ip TCP UDP ICMP เเละ assign ไปที่ processed_data
```
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

สร้าง array ขึ้นมา สำหรับ เก็บ ip ของ เเต่ละ service คือ TCP UDP ICMP
```
- name: Initialize protocol-specific arrays
  set_fact:
    TCP: []
    UDP: []
    ICMP: []
```
จะทำการ nested loop processed_data เเละ เข้าถึง src_up dst_up TCP เเละ เก็บไว้ในตัวเเปล TCP เเละ ทำเเบบนี้กับ UDP ICMP 	 
```
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

```

ทำการ loop เข้าไปใน TCP เเละ add_host เป็นทำ เเบบนี้ ทั้ง TCP UDP ICMP
```
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

<br/>
<br/>
<br/>

__main.yaml (หลังจาก setup inventory เเล้ว)__

filter เฉพาะ host group ที่ หลังท้ายด้วย _TCP เท่านนั้น เเละเรียก tasks check_tcp.yaml
```
- name: Configure TCP servers
  hosts: "*_TCP"
  gather_facts: no
  tasks:
    - import_tasks: tasks/check_tcp.yaml
```

__check_tcp.yaml__

print เช็ค host ที่เข้ามา
```
- name: Debug TCP server details
  debug:
    msg: |
      Host: {{ inventory_hostname }}
      IP: {{ ansible_host }}
      Port: {{ ansible_port | default('not set') }}
      TCP Port: {{ hostvars[inventory_hostname].tcp_port | default('not set') }}
      Groups: {{ group_names }}
    verbosity: 0
```
เช็คว่า มี tcp_port อยุ่ใน host ไหม
```
- name: Check if tcp_port is defined
  ansible.builtin.assert:
    that:
      - "tcp_port is defined"
    fail_msg: "tcp_port is not defined for host {{ inventory_hostname }}"
    quiet: true
```

รัน คำสั่ง nc -lvp { port } เพื่อเปิด port 
```
- name: Run nc -lvp on TCP server port
  ansible.builtin.command:
    cmd: "nc -lvp {{ tcp_port }}"
  async: 3600  # 1 hour 
  poll: 0      # background process
  register: nc_result
  failed_when: false
```

เช็คว่า port ถูกเปิดขึ้นเเล้ว ด้วย netstat เเละ print nc_result 
```
- name: Check if TCP port is open using netstat
  ansible.builtin.command:
    cmd: "netstat -tuln | grep :{{ tcp_port }}"
  register: netstat_result
  failed_when: false

- name: Debug raw nc_result output
  debug:
    var: nc_result

- name: Debug TCP server setup results
  debug:
    msg: |
      TCP Server Setup Results:
      Host: {{ inventory_hostname | default('unknown') }}
      IP: {{ ansible_host | default('unknown') }}
      TCP Port: {{ tcp_port | default('unknown') }}
      Command: nc -lvp {{ tcp_port }}
      Status: {{ 'Success' if nc_result.ansible_job_id is defined else 'Failed' }}
      Job ID: {{ nc_result.ansible_job_id | default('Not available') }}
    verbosity: 1
```
__**ทำเเบบเดียวกัน กับ UDP ด้วย เเต่ จะเปลียน nc -lvp เป็น nc -ulvp {{ udp_port }}__

<br/>
<br/>
<br/>

__main.yaml (หลังจาก check_tcp, check_udp เสร็จ)__

fliter เเค่ __client จิงๆๆ version ก่อนจะมี server ด้วยเลย เเต่ code เดิมมัน work เลยไม่ยุ่งกับมัน เเละ สร้าง ตัว เเปล is_tcp_client เพื่อเช็คว่า host นี้เป็น TCP หรือป่าว เเละ เรียกใช้ connect_client หลังจาก task connect client เสร็จ ก็จะ เก็บ result ไว้ใน conectivity_results
```
- name: Test clients TCP
  hosts: "*_client_*"
  gather_facts: yes
  vars:
    is_tcp_client: "{{ inventory_hostname in (groups | dict2items | selectattr('key', 'match', '.*_TCP$') | map(attribute='value') | flatten) }}"
  tasks:
    - import_tasks: tasks/connect_client.yaml
      when: is_tcp_client | bool
    - name: Store TCP connectivity result
      set_fact:
        connectivity_results: "{{ (hostvars['localhost']['connectivity_results'] | default([])) + [{
          'SRC': src_name, 
          'src_ip': ansible_host, 
          'DST': dst_name, 
          'dst_ip': dst_ip, 
          'protocol': 'TCP', 
          'port': tcp_port | default(''), 
          'success': (nc_result.rc == 0) if nc_result is defined else false,
          'error_msg': (nc_result.stderr | default('') if nc_result.rc != 0 else '') if nc_result is defined else 'Test not executed',
          'timestamp': ansible_date_time.iso8601 | default('')
        }] }}"
      delegate_facts: true
      delegate_to: localhost
      when: is_tcp_client | bool
```

__connect_client.yaml__

Filter client เเละ print host เเละ check ว่ามี dst_ip มาด้วยไหมถ้า ไม่มีให้ skip
```
- name: Filter for TCP clients only
  ansible.builtin.set_fact:
    is_client: "{{ 'client' in inventory_hostname }}"
  when: "'dst_ip' in hostvars[inventory_hostname]"

- name: Debug TCP client details
  debug:
    msg: |
      Client Host: {{ inventory_hostname }}
      Source IP: {{ ansible_host }}
      Destination IP: {{ dst_ip }}
      TCP Port: {{ tcp_port | default('not set') }}
      Groups: {{ group_names }}
  when: is_client | bool

- name: Check if dst_ip is defined
  ansible.builtin.assert:
    that:
      - "dst_ip is defined"
    fail_msg: "dst_ip is not defined for host {{ inventory_hostname }}"
    quiet: true
  when: is_client | bool

- name: Skip if tcp_port is not defined
  set_fact:
    skip_tcp_test: "{{ tcp_port is not defined }}"
  when: is_client | bool
```

รันคำสั่ง nc -z dst_ip tcp_port เพื่อเช็คว่า port นั้นๆเปิดอยุ่หรือป่าว เเละ เก็บ result ไว้ใน connectivity_results ทำเเบบเดียวกัน กับ UDP เเต่ว่า เปลียน command เป็น "nc -z -u -v {{ dst_ip }} {{ udp_port }}"
```
- name: Run nc -z to test connectivity to dst_ip on tcp_port
  ansible.builtin.command:
    cmd: "nc -z -v {{ dst_ip }} {{ tcp_port }}"
  register: nc_result
  failed_when: false  
  when: is_client | bool and not (skip_tcp_test | default(false))

- name: Set test timestamp
  ansible.builtin.set_fact:
    test_timestamp: "{{ ansible_date_time.iso8601 | default('') }}"
  when: is_client | bool

- name: Display nc -z result
  debug:
    msg: |
      TCP Connectivity test from {{ inventory_hostname }} to {{ dst_ip }}:{{ tcp_port }}:
      Status: {{ 'Success' if nc_result.rc == 0 else 'Failed' }}
      Exit Code: {{ nc_result.rc }}
      Source: {{ src_name | default(inventory_hostname) }} ({{ ansible_host }})
      Destination: {{ dst_name | default('unknown') }} ({{ dst_ip }})
      Port: {{ tcp_port }}
      Command: nc -z -v {{ dst_ip }} {{ tcp_port }}
      Timestamp: {{ test_timestamp }}
      Output: {{ nc_result.stdout_lines | default(['No output']) | to_nice_json }}
      Error (if any): {{ nc_result.stderr_lines | default(['No error']) | to_nice_json }}
    verbosity: 0
  when: is_client | bool and not (skip_tcp_test | default(false))

- name: Store TCP connectivity result
  delegate_to: localhost
  ansible.builtin.set_fact:
    connectivity_results: "{{ connectivity_results | default([]) + [{
      'SRC': src_name | default(inventory_hostname | regex_replace('_client_.*', '')),
      'src_ip': ansible_host,
      'DST': dst_name | default('unknown'),
      'dst_ip': dst_ip,
      'protocol': 'TCP',
      'port': tcp_port | default(''),
      'success': nc_result.rc == 0,
      'error_msg': nc_result.stderr | default(''),
      'timestamp': test_timestamp
    }] }}"
  when: is_client | bool and not (skip_tcp_test | default(false))
```


__main.yaml (หลังจากทำ connect_client เสร็จ)__

Filter เฉพาะ host ที่มี ICMP ต่อท้าย เเละ เรียกใช้ check_icmp.yaml เเละ เก็บ result ไว้ใน connectivity_results
```
- name: Configure ICMP servers
  hosts: "*_ICMP"
  gather_facts: no
  tasks:
    - import_tasks: tasks/check_icmp.yaml
    - name: Store ICMP connectivity result
      set_fact:
        connectivity_results: "{{ (hostvars['localhost']['connectivity_results'] | default([])) + [{
          'SRC': src_name, 
          'src_ip': ansible_host, 
          'DST': dst_name, 
          'dst_ip': dst_ip, 
          'protocol': 'ICMP', 
          'port': '', 
          'success': (ping_test_result.rc == 0) if ping_test_result is defined else false,
          'error_msg': (ping_test_result.stderr | default('') if ping_test_result.rc != 0 else '') if ping_test_result is defined else 'Test not executed',
          'timestamp': ansible_date_time.iso8601 | default('')
        }] }}"
      delegate_facts: true
      delegate_to: localhost
```

__check_icmp.yaml__

print ICMP เเละ เช็คว่า มี dst_ip อยุ่หรือป่าว เเละ รันคำสั่ง ping -c 4 dst_ip เเละ เก็บ result ไว้ใน ping_test_result
```
- name: Debug ICMP server details
  debug:
    msg: |
      Host: {{ inventory_hostname | default('unknown') }}
      IP: {{ ansible_host | default('unknown') }}
      Port: {{ ansible_port | default('not set') }}
      Destination IP: {{ hostvars[inventory_hostname].dst_ip | default('not set') }}
      Groups: {{ group_names | default([]) }}
    verbosity: 0

- name: Check if dst_ip is defined
  ansible.builtin.assert:
    that:
      - "dst_ip is defined"
    fail_msg: "dst_ip is not defined for host {{ inventory_hostname | default('unknown') }}"
    quiet: true

- name: Ensure ICMP responsiveness on server
  ansible.builtin.command:
    cmd: "ping -c 4 {{ dst_ip | default('127.0.0.1') }}"
  register: ping_test_result
  failed_when: false

- name: Debug ICMP test results
  debug:
    msg: |
      ICMP Test Results:
      Source: {{ inventory_hostname | default('unknown') }}
      Destination: {{ dst_ip | default('unknown') }}
      Success: {{ (ping_test_result.rc == 0) if ping_test_result is defined else 'unknown' }}
      Output: {{ ping_test_result.stdout | default('No output') }}
      Error: {{ ping_test_result.stderr | default('No error') }}
    verbosity: 1
```


__main.yaml (หลังจากทำ check_icmp เสร็จ)__

```
- name: Save results to CSV
  hosts: localhost
  connection: local
  gather_facts: no
  tasks:
    - name: Create new connectivity results CSV
      ansible.builtin.copy:
        content: |
          "SRC","src_ip","DST","dst_ip","protocol","port","success","error_msg","timestamp"
          {% for result in connectivity_results %}
          "{{ result.SRC }}","{{ result.src_ip }}","{{ result.DST }}","{{ result.dst_ip }}","{{ result.protocol }}","{{ result.port }}","{{ result.success }}","{{ result.error_msg | default('') | replace('"', '""') }}","{{ result.timestamp | default('') }}"
          {% endfor %}
        dest: "connectivity_results.csv"

    - name: Print detailed connectivity results
      debug:
        var: connectivity_results
        verbosity: 0
        
    - name: Print formatted connectivity results
      debug:
        msg: |
          ==========================================
          DETAILED CONNECTIVITY RESULTS
          ==========================================
          {% for result in connectivity_results %}
          {{ loop.index }}. {{ result.SRC }} ({{ result.src_ip }}) to {{ result.DST }} ({{ result.dst_ip }}) via {{ result.protocol }}{% if result.port %} on port {{ result.port }}{% endif %}
             Status: {{ 'SUCCESS' if result.success else 'FAILED' }}
             {% if not result.success %}Error: {{ result.error_msg | default('Unknown error') }}{% endif %}
             Timestamp: {{ result.timestamp }}
          {% else %}
          No connectivity results found.
          {% endfor %}
        verbosity: 0

    - name: Generate connectivity test summary
      debug:
        msg: |
          ==========================================
          CONNECTIVITY TEST SUMMARY
          ==========================================
          Total Tests: {{ connectivity_results | length }}
          Successful Tests: {{ connectivity_results | selectattr('success', 'equalto', true) | list | length }}
          Failed Tests: {{ connectivity_results | selectattr('success', 'equalto', false) | list | length }}
          
          FAILED CONNECTIONS:
          {% for result in connectivity_results if not result.success %}
          - {{ result.SRC }} ({{ result.src_ip }}) to {{ result.DST }} ({{ result.dst_ip }}) via {{ result.protocol }}{% if result.port %} on port {{ result.port }}{% endif %}
            Error: {{ result.error_msg | default('Unknown error') }}
          {% else %}
          No failed connections.
          {% endfor %}
          
          SUCCESSFUL CONNECTIONS:
          {% for result in connectivity_results if result.success %}
          - {{ result.SRC }} ({{ result.src_ip }}) to {{ result.DST }} ({{ result.dst_ip }}) via {{ result.protocol }}{% if result.port %} on port {{ result.port }}{% endif %}
          {% else %}
          No successful connections.
          {% endfor %}
        verbosity: 0
```