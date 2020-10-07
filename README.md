# Ansible Cheat List

## debugs

Print a variable (here print all default variables)
```
- hosts: k8s_hosts
  tasks:
    - debug:
        var: hostvars
```
print content and vars with msg
```
    - name: "loop through list"
      debug:
        msg: "{{ (item | regex_replace('(^.*)s$', '\\1')) }}"   
```

## File 

read file from remote host and modify it with replace
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

copy file (here remote to remote)

```
    - name: copy kubernetes admin.conf to root home dir
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/ubuntu/.kube/config
        remote_src: yes
        owner: ubuntu
```

Create a file with some content (here remote to remote)
```
    - name: Create docker daemon file settings (systemd cgroup)
      copy:
        dest: "/etc/docker/daemon.json"
        content: |
          {
            "exec-opts": ["native.cgroupdriver=systemd"],
            "log-driver": "json-file",
            "log-opts": {
              "max-size": "100m"
          },
          "storage-driver": "overlay2"
          }
```
Create a directory
```
    - name: Create .kube in ubuntu home
      become: no
      file:
        path: /home/ubuntu/.kube
        state: directory
        mode: 0755
```
Create a file based on curl equivalent of  "curl https://docs.projectcalico.org/manifests/calico.yaml -O"
```
    - name: Get Calico manifest for installation
      uri:
        url: 'https://docs.projectcalico.org/manifests/calico.yaml'
        method: GET
        creates: '/home/ubuntu/calico.yaml'
        dest: '/home/ubuntu/calico.yaml'
```

## Kernel/Systemd

restart service (systemctl start docker)
```
    - name: restart docker service
      systemd:
        name: docker
        daemon_reload: yes
        state: restarted
```

load kernel module

```
    - name: load br_netfilter module
      modprobe:
        name: br_netfilter
        state: present
```

change systctl value variables + reload sysctl files

```
    - name: set sysctl for iptables
      sysctl:
        name: net.bridge.bridge-nf-call-iptables
        value: 1
        sysctl_set: yes
        state: present
        sysctl_file: /etc/sysctl.d/k8s.conf
        reload: yes
``` 

## packages

pip
```
    - name: install pexpect
      pip:
        name: pexpect
      become: yes
```
apt
```
    - name: Add apt key from google to sources
      apt_key:
        url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
        state: present            
    - name: Add k8S (xenial still the latest) repository into apt sources
      apt_repository:
        repo: deb https://apt.kubernetes.io/ kubernetes-xenial main
        state: present
        filename: kubernetes
    - name: Run the equivalent of "apt-get update" as a separate step
      apt:
        update_cache: yes
    - name: Install kubeadm kubelet and kubectl
      apt:
        name: 
          - kubelet
          - kubeadm
          - kubectl
        update_cache: yes
        state: present
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

## Loops & conditions

loops:
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
conditions

```
    - name: Get the cluster token (executed on master only)
      shell: kubeadm token list | grep bootstrap | awk {'print $1'}
      register: token
      become: false
      when: "'masters' in {{group_names}}"
 ```
## workflow / server interaction

expect
```
    - name: Set password if expired (happens after fresh reinstall)
      delegate_to: 127.0.0.1
      ignore_errors: yes 
      expect:
        command: "ssh {{ ansible_ssh_common_args }} {{ ansible_user }}@{{ inventory_hostname }}"
        timeout: 10
        responses:
          "password:": "{{ ssh_command_pass_old }}"
          "Current password:": "{{ ssh_command_pass_old }}"
          "New password:": "{{ ssh_command_pass }}"
          "Retype new password:": "{{ ssh_command_pass }}"
#          # if succesfully login then quit
          "\\~\\]\\$": exit
```
command
```
    - name: Initialize the Kubernetes cluster using kubeadm
      command: kubeadm init --pod-network-cidr=10.244.0.0/16
      register: status
    - debug: 
        var: status
```

Reboot and wait for resurrection

```
    - name: Attempting reboot
      shell: reboot
      async: 1200
      poll: 0
    - name: Waiting for resurection
      wait_for_connection:
        delay: 60
        timeout: 300
```
shell (even dirtier than command), but that's when you need some pipes
```
    - name: Get the cluster token 
      shell: kubeadm token list | grep bootstrap | awk {'print $1'}
      register: token
      become: false
      when: "'masters' in {{group_names}}"
```
