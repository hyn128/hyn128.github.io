---
title: Ansible Facts와 Linux Server 점검 자동화
description: Ansible Facts로 여러 Managed Node의 System 정보를 수집하고 dstat, iostat, vmstat와 df 결과를 안전하게 File로 저장하는 Playbook을 구성합니다
date: 2026-10-09
series: CloudNative
tags:
  - CloudNative
  - AutoEverSW
  - Ansible
---

## 요약

---

> Ansible Control Node에서 여러 Managed Node의 Facts를 수집해 OS, Kernel, CPU, Memory, Interface와 Mount 정보를 같은 형식으로 기록합니다. 추가로 `dstat`, `iostat`, `vmstat`와 `df`를 실행해 점검 시점의 System 상태를 저장합니다. 반복 실행 때 무조건 내용을 누적하는 `shell` Redirect 대신 `command`로 측정하고 `copy`로 Report를 갱신합니다.

## 1. 작업 범위와 실행 위치

---

[Ansible Architecture와 Inventory 구성]({% post_url 2026-09-22-cloud-native-58-ansible-architecture-inventory %})에서 Control Node, SSH와 Inventory를 구성했고, [Ansible 변수와 Vault·Facts·제어문]({% post_url 2026-09-23-cloud-native-60-ansible-variables-vault-facts-control-flow %})에서 Facts와 조건문을 다뤘습니다. 이번 글에서는 준비된 Inventory를 이용한 Server 점검에 집중합니다.

| 표기 | 실행 위치 | 작업 |
| --- | --- | --- |
| `[CONTROL]` | Ansible Control Node | Playbook 작성, 문법 검사와 실행 |
| `[MANAGED]` | Managed Node | Facts 제공, 점검 명령 실행과 Report 저장 |

```text
[CONTROL] Inventory와 Playbook
  ↓ SSH
[MANAGED] Facts 수집·점검 명령
  ↓
/var/log/daily_check/*.log
```

이 방식은 특정 시점의 점검 Report를 만드는 자동화입니다. 지속적인 시계열 Monitoring과 Alert는 Prometheus, CloudWatch 같은 Monitoring System을 사용합니다.

## 2. Project 구조와 Inventory

---

```text
ansible-monitoring/
├── ansible.cfg
├── inventory.ini
├── monitoring_facts.yml
├── monitoring_system.yml
└── vars_packages.yml
```

```ini
[monitoring_targets]
server-a ansible_host=<SERVER_A_IP>
server-b ansible_host=<SERVER_B_IP>

[monitoring_targets:vars]
ansible_user=<MANAGED_USER>
```

대상 Host를 확인합니다.

```bash
# [CONTROL]
ansible-inventory -i inventory.ini \
  --graph

ansible monitoring_targets \
  -i inventory.ini \
  -m ansible.builtin.ping
```

## 3. Facts Report Playbook

---

Facts는 Play 시작 시 Managed Node에서 수집한 System 정보입니다. `monitoring_facts.yml`을 작성합니다.

```yaml
{% raw %}
---
- name: Write system facts report
  hosts: monitoring_targets
  gather_facts: true
  become: true

  vars:
    log_directory: /var/log/daily_check

  tasks:
    - name: Create report directory
      ansible.builtin.file:
        path: "{{ log_directory }}"
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Write facts report
      ansible.builtin.copy:
        dest: "{{ log_directory }}/system_info.log"
        owner: root
        group: root
        mode: "0644"
        content: |
          Date: {{ ansible_facts.date_time.iso8601 }}
          Hostname: {{ ansible_facts.hostname }}
          OS: {{ ansible_facts.distribution }}
          OS Version: {{ ansible_facts.distribution_version }}
          Kernel: {{ ansible_facts.kernel }}
          CPU vCPUs: {{ ansible_facts.processor_vcpus }}
          Memory MiB: {{ ansible_facts.memory_mb.real.total }}
          Interfaces: {{ ansible_facts.interfaces | join(', ') }}
          IPv4 Addresses: {{ ansible_facts.all_ipv4_addresses | join(', ') }}
          Mounts:
          {{ ansible_facts.mounts | to_nice_yaml(indent=2) | indent(10, true) }}
{% endraw %}
```

| Fact | 의미 |
| --- | --- |
| `date_time.iso8601` | Facts 수집 시각 |
| `hostname` | Managed Node Hostname |
| `distribution*` | OS 배포판과 Version |
| `kernel` | Kernel Version |
| `processor_vcpus` | 사용 가능한 vCPU 수 |
| `memory_mb.real.total` | 전체 Physical Memory MiB |
| `interfaces` | Network Interface 이름 |
| `all_ipv4_addresses` | 수집된 IPv4 주소 |
| `mounts` | Mount Point, Device와 용량 정보 |

`ansible_date_time` 계열 값은 Play 시작 시 Facts를 수집한 시각입니다. 장시간 실행하는 Play에서 현재 시각이 계속 갱신되는 값으로 사용하지 않습니다.

## 4. Facts Playbook 검증과 실행

---

```bash
# [CONTROL]
ansible-playbook \
  -i inventory.ini \
  --syntax-check \
  monitoring_facts.yml

ansible-playbook \
  -i inventory.ini \
  --check --diff \
  monitoring_facts.yml \
  -K

ansible-playbook \
  -i inventory.ini \
  monitoring_facts.yml \
  -K
```

`--syntax-check`는 YAML과 Playbook 구조를 확인합니다. `--check`는 변경을 수행하지 않고 지원 Module이 예상 변경을 보고합니다. `--diff`에는 File 내용이 표시될 수 있으므로 Secret이 포함된 Template에는 사용하지 않습니다. `-K`는 Managed Node의 Privilege Escalation Password를 요청합니다.

Report를 확인합니다.

```bash
# [CONTROL]
ansible monitoring_targets \
  -i inventory.ini \
  -b \
  -m ansible.builtin.command \
  -a 'cat /var/log/daily_check/system_info.log'
```

## 5. Monitoring Package 변수

---

`vars_packages.yml`에서 OS Family별 Package와 `dstat` 명령 이름을 구분합니다.

```yaml
---
log_directory: /var/log/daily_check

monitoring_packages:
  Debian:
    - dstat
    - sysstat
  RedHat:
    - pcp-system-tools
    - sysstat

dstat_binary:
  Debian: dstat
  RedHat: pcp-dstat
```

Package 이름은 배포판과 Version에 따라 다를 수 있습니다. 실행 전에 대상 Repository에서 `dstat` 또는 `pcp-dstat`, `sysstat` Package 제공 여부를 확인합니다.

## 6. System 점검 Playbook

---

`monitoring_system.yml`을 작성합니다. `shell` Redirect를 반복하는 대신 각 명령의 Standard Output을 등록하고 File 내용을 한 번에 갱신합니다.

```yaml
{% raw %}
---
- name: Collect point-in-time system report
  hosts: monitoring_targets
  gather_facts: true
  become: true

  vars_files:
    - vars_packages.yml

  pre_tasks:
    - name: Validate supported OS family
      ansible.builtin.assert:
        that:
          - ansible_facts.os_family in monitoring_packages
        fail_msg: >-
          Unsupported OS family: {{ ansible_facts.os_family }}

  tasks:
    - name: Create report directory
      ansible.builtin.file:
        path: "{{ log_directory }}"
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Install monitoring packages on Debian family
      ansible.builtin.apt:
        name: "{{ monitoring_packages.Debian }}"
        state: present
        update_cache: true
      when: ansible_facts.os_family == "Debian"

    - name: Install monitoring packages on Red Hat family
      ansible.builtin.dnf:
        name: "{{ monitoring_packages.RedHat }}"
        state: present
      when: ansible_facts.os_family == "RedHat"

    - name: Collect dstat samples
      ansible.builtin.command:
        argv:
          - "{{ dstat_binary[ansible_facts.os_family] }}"
          - --nocolor
          - "1"
          - "5"
      register: dstat_result
      changed_when: false
      when: not ansible_check_mode

    - name: Collect iostat report
      ansible.builtin.command:
        argv:
          - iostat
          - -t
          - -c
          - -d
      register: iostat_result
      changed_when: false
      when: not ansible_check_mode

    - name: Collect vmstat report
      ansible.builtin.command:
        argv:
          - vmstat
          - -d
          - -t
      register: vmstat_result
      changed_when: false
      when: not ansible_check_mode

    - name: Collect file system report
      ansible.builtin.command:
        argv:
          - df
          - -h
      register: df_result
      changed_when: false
      when: not ansible_check_mode

    - name: Write dstat report
      ansible.builtin.copy:
        dest: "{{ log_directory }}/dstat.log"
        content: "{{ dstat_result.stdout }}\n"
        owner: root
        group: root
        mode: "0644"
      when: not ansible_check_mode

    - name: Write iostat report
      ansible.builtin.copy:
        dest: "{{ log_directory }}/iostat.log"
        content: "{{ iostat_result.stdout }}\n"
        owner: root
        group: root
        mode: "0644"
      when: not ansible_check_mode

    - name: Write vmstat report
      ansible.builtin.copy:
        dest: "{{ log_directory }}/vmstat.log"
        content: "{{ vmstat_result.stdout }}\n"
        owner: root
        group: root
        mode: "0644"
      when: not ansible_check_mode

    - name: Write file system report
      ansible.builtin.copy:
        dest: "{{ log_directory }}/df.log"
        content: "{{ df_result.stdout }}\n"
        owner: root
        group: root
        mode: "0644"
      when: not ansible_check_mode
{% endraw %}
```

`command` Module은 Shell을 통하지 않으므로 Redirect와 Pipe를 해석하지 않습니다. 이 Playbook은 `stdout`을 등록한 뒤 `copy` Module로 File을 관리합니다. 점검 명령은 상태를 읽기만 하므로 `changed_when: false`로 표시합니다.

## 7. 실행과 결과 해석

---

```bash
# [CONTROL]
ansible-playbook \
  -i inventory.ini \
  --syntax-check \
  monitoring_system.yml

ansible-playbook \
  -i inventory.ini \
  --check --diff \
  monitoring_system.yml \
  -K

ansible-playbook \
  -i inventory.ini \
  monitoring_system.yml \
  -K
```

Check Mode에서는 Package와 Directory 변경 예상만 확인하고 실제 측정 명령과 Report 작성은 건너뜁니다. 실제 실행 후 Managed Node에 다음 File이 생성됩니다.

```text
/var/log/daily_check/
├── system_info.log
├── dstat.log
├── iostat.log
├── vmstat.log
└── df.log
```

Play Recap에서 `unreachable`과 `failed`를 먼저 확인합니다. 특정 Host만 점검하려면 `--limit <HOST_OR_GROUP>`을 사용합니다.

## 8. 자동화의 한계

---

이 Playbook은 실행 시점의 Snapshot을 여러 Host에서 같은 형식으로 수집하는 데 적합합니다. 다음 기능은 제공하지 않습니다.

- 시간에 따른 Metric 저장과 Query

- 실시간 Dashboard

- Threshold 기반 Alert와 Notification

- Trace와 Application Request 상관관계 분석

지속 Monitoring은 Node Exporter와 Prometheus 또는 CloudWatch Agent 같은 수집기를 사용하고, Ansible은 수집기 설치·설정과 일회성 점검 자동화에 사용합니다.

> **최종 정리**
> - Facts는 Managed Node에서 수집한 System 정보이며 Host별 Report를 같은 형식으로 만들 수 있습니다.
>
> - 반복 Redirect 대신 `command` 결과를 등록하고 `copy`로 Report를 관리합니다.
>
> - Check Mode가 모든 Module과 명령 실행을 완전히 검증하는 것은 아닙니다.
>
> - Package 이름과 Block Device 이름은 OS와 환경마다 다르므로 고정하지 않습니다.
>
> - Ansible 점검은 Snapshot 자동화이며 지속 Monitoring System을 대체하지 않습니다.

## 참고 자료

---

- [Ansible Facts](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_vars_facts.html)

- [gather_facts Module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/gather_facts_module.html)

- [Check Mode와 Diff Mode](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_checkmode.html)
