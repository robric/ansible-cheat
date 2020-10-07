# Ansible Cheat List

##debugs

Print a variable (here print all default variables)
```
- hosts: k8s_hosts
  tasks:
    - debug:
        var: hostvars
```

## File 

### read file from remote host and modify it with replace

```
    - name: Get boot command file for to see if modification is required
      slurp: 
        src: "/boot/firmware/cmdline.txt"
      register: boot_command_file
    - name: Modify boot command with cgroup settings
      replace:
        path: /boot/firmware/cmdline.txt
        regexp: '^(net(.*)$)'
        replace: '\1 cgroup_memory=1 cgroup_enable=memory'
      when: "'cgroup' not in (boot_command_file['content'] | b64decode)"
      become: true
```

## Kernel/Systemd

## packages

pip
```
    - name: install pexpect
      pip:
        name: pexpect
      become: yes
```

## filter / regexps: \1 repeats matched pattern
```
- hosts: k8s_hosts
  gather_facts: no
  tasks:
    - name: "loop through list"
      debug:
        msg: "{{ (item | regex_replace('(^.*)s$', '\\1')) }}"
      when: item in ["workers","masters"]
      with_items: "{{group_names}}"
```

## Loops

```
- name: "loop through list"
  debug:
    msg: "An item: {{ item }}"
  with_items:
    - 1
    - 2
    - 3

    - name: "loop through list group_names default variable"
      debug:
        msg: "An item: {{ item }}"
      with_items: "{{group_names}}"

 ```
