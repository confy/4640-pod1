## ACIT Lab 4 - Adrian McFarlane

Both VMs are on AWS. I am using Ubuntu 22.04 and RHEL 8
### Commands:
Ansible Inventory
```sh
ansible-inventory --graph
```
<!-- Create embed image -->
![Ansible Inventory](https://cdn.discordapp.com/attachments/891094743969329152/1035955297786806342/unknown.png)

Install Podman module in ansible

```sh
ansible-galaxy collection install containers.podman
```

Run a Playbook

```sh
ansible-playbook -i inventory/hosts.aws_ec2.yml playbookX.yml
```

`ansible.cfg`

```ini
[defaults]
inventory = inventory

[inventory]
enabled_plugins = ini, aws_ec2

```

`inventory/hosts.aws_ec2.yml`

```yml
---
plugin: aws_ec2
regions:
  - us-west-2

strict: False
filters:
  instance-state-name: running
keyed_groups:
  - prefix: tag
    key: tags
compose:
  ansible_host: public_ip_address
```

Group names are defined by AWS name tags and are used to define the user to connect with by distro:
After the first connection this can be changed to "app" user

`group_vars/tag_Name_acit4640_rhel.yml`

```yml
---
ansible_user: ec2-user
ansible_ssh_private_key_file: ./key.pem
```

`group_vars/tag_Name_acit4640_ubuntu.yml`

```yml
---
ansible_user: ubuntu
ansible_ssh_private_key_file: ./key.pem
```

`playbook1.yml`

```yml
---
- hosts: aws_ec2

  tasks:
    - name: Create a user
      become: yes
      user:
        name: "app"
        shell: "/bin/bash"
        state: present

    - name: add user to sudo group
      when: inventory_hostname in groups['tag_Name_acit4640_ubuntu']
      become: yes
      user:
        name: "app"
        groups: "sudo"
        append: yes

    - name: Add user to wheel group
      when: inventory_hostname in groups['tag_Name_acit4640_rhel']
      become: yes
      user:
        name: "app"
        groups: "wheel"
        append: yes

    - name: Copy .ssh directory
      become: yes
      copy:
        src: "{{ ansible_env.HOME }}/.ssh"
        dest: "/home/app"
        owner: "app"
        group: "app"
        mode: "0755"
        remote_src: yes

    - name: Configure sshd
      become: yes
      lineinfile:
        path: "/etc/ssh/sshd_config"
        regex: "^(#)?{{item.key}}"
        line: "{{item.key}} {{item.value}}"
        state: present
      loop:
        - { key: "PermitRootLogin", value: "no" }
        - { key: "PasswordAuthentication", value: "no" }
      register: ssh_config

    - name: Restart ssh
      service:
        name: sshd
        state: restarted
      when: ssh_config.changed
```

`playbook2.yml`

```yml
---
- hosts: aws_ec2

  tasks:
    - name: Install podman with dnf
      when: inventory_hostname in groups['tag_Name_acit4640_rhel']
      become: yes
      dnf:
        update_cache: yes
        name: podman
        state: present

    - name: Install podman with apt
      when: inventory_hostname in groups['tag_Name_acit4640_ubuntu']
      become: yes
      apt:
        update_cache: yes
        name: podman
        state: present

    - name: Run container
      containers.podman.podman_container:
        name: nginx
        image: docker.io/library/nginx
        state: started
        ports:
          - "8080:80"
```
