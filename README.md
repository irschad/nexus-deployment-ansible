# Automate Nexus Deployment

## Project Overview
This project demonstrates the automation of Nexus deployment on an EC2 instance using Ansible. The goal is to provision a server, configure it, and ensure Nexus is running successfully.

---

## Key Features
- Automated EC2 instance configuration.
- Installation and setup of Nexus Repository Manager.
- Verification of successful deployment using Ansible tasks.

---

## Technologies
- **Ansible**: For automation and configuration management.
- **Nexus**: Repository management system.
- **AWS EC2**: Virtual server hosting.
- **Java**: Required runtime environment for Nexus.
- **Linux**: Operating system for server configuration.

---

## Project Workflow
1. **Provision an EC2 Instance**
   - Create an EC2 instance using AWS Management Console or CLI.
   - Use the existing SSH key for secure access.
   - Ensure the instance is accessible via its public IP.

2. **Write Ansible Playbook**
   - Configure server environment (install Java, net-tools, etc.).
   - Download and unpack the Nexus installer.
   - Create a dedicated Linux user for Nexus.
   - Set appropriate permissions for Nexus directories.
   - Start Nexus and verify it is running.

3. **Verification**
   - Ensure Nexus is successfully started and accessible on port 8081.
   - Use Ansible tasks to confirm process and port status.

---

## Ansible Configuration Files

### 1. `deploy-nexus.yaml`
This playbook automates the entire Nexus installation and configuration process:

```yaml
---
- name: Install Java and net-tools
  hosts: nexus_server
  tasks: 
    - name: yum update
      yum: update_cache=yes update_only=yes  
    - name: Install Java 11
      yum:
        name: java-11-amazon-corretto
        state: present      
    - name: Install net-tools
      yum:
        name: net-tools
        state: present

- name: Download and unpack Nexus installer
  hosts: nexus_server
  tasks:
    - name: Check nexus folder stats
      stat:
        path: /opt/nexus
      register: stat_result
    - name: Download Nexus
      get_url:
        url: https://download.sonatype.com/nexus/3/latest-unix.tar.gz
        dest: /opt/
      register: download_result
    - name: Untar Nexus installer
      unarchive:
        src: "{{download_result.dest}}"
        dest: /opt
        remote_src: True
      when: not stat_result.stat.exists
    - name: Find nexus folder
      find:
        paths: /opt
        pattern: "nexus-*"
        file_type: directory
      register: find_result
    - name: Rename nexus folder
      shell: mv {{find_result.files[0].path}} /opt/nexus
      when: not stat_result.stat.exists

- name: Create nexus user to own nexus folder
  hosts: nexus_server
  tasks:
    - name: Ensure group nexus exists
      group:
        name: nexus
        state: present
    - name: Create nexus user 
      user:
        name: nexus
        group: nexus
    - name: Make nexus user owner of nexus folder
      file:
        path: /opt/nexus
        state: directory
        owner: nexus
        group: nexus
        recurse: yes
    - name: Make nexus user owner of sonatype-work folder
      file:
        path: /opt/sonatype-work
        state: directory
        owner: nexus
        group: nexus
        recurse: yes

- name: Start nexus with nexus user
  hosts: nexus_server
  become: True
  become_user: nexus
  tasks:
    - name: Set run_as_user nexus
      lineinfile: 
        path: /opt/nexus/bin/nexus.rc
        regexp: '^#run_as_user=""'
        line: run_as_user="nexus"
    - name: Start nexus
      command: /opt/nexus/bin/nexus start

- name: Verify nexus is running
  hosts: nexus_server
  tasks:
    - name: Check with ps
      shell: ps aux | grep nexus
      register: app_status
    - debug: msg={{app_status.stdout_lines}}
    - name: Wait for 90 seconds
      pause:
        seconds: 90 
    - name: Check with netstat
      shell: netstat -plnt
      register: app_status
    - debug: msg={{app_status.stdout_lines}}
```

### 2. `ansible.cfg`
```ini
[defaults]
host_key_checking = False
```

### 3. `hosts` file
```ini
[nexus_server]
3.228.25.53 ansible_python_interpreter=/usr/bin/python3.9  ansible_ssh_private_key_file=~/.ssh/id_rsa ansible_user=ec2-user
```
---

## Steps to Run the Project
1. Clone the repository or copy the project files to your local machine.
2. Ensure your Ansible control node can connect to the EC2 instance.
3. Update the `hosts` file with the public IP of your EC2 instance.
4. Run the playbook:
   ```bash
   ansible-playbook -i hosts deploy-nexus.yaml
   ```
5. Verify that Nexus is running by accessing `http://3.228.25.53:8081` in your browser.

---

## Conclusion
This project simplifies the Nexus deployment process by automating server setup and configuration. Using Ansible ensures consistency and repeatability, making it a valuable tool for DevOps automation.
