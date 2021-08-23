# Docker Container with simple Node.js Hello World application 

The goal of this example is to show you how to get a Node.js application into a Docker container. The guide is intended for development, and not for a production deployment. The guide assumes you have a basic understanding of how a Node.js application is structured.
We will create a simple web application in Node.js, then we will build a Docker image for that application, and lastly we will instantiate a container from that image.

## Prerequisites
- Three VPS/Dedicated Servers
- SSH root access to VPS/Dedicated Servers

## About the playbook

~~~
---
- name: "Docker Image/Build and Image/Push"
  hosts: build
  become: true
  vars_files:
    - docker.vars

  tasks:

    - name: Update all packages to their latest version
      apt:
        name: "*"
        state: latest


    - name: "Install packages to allow apt to use a repository over HTTPS"
      apt:
        pkg:
        - apt-transport-https
        - ca-certificates
        - curl
        - gnupg
        - lsb-release
        state: present
        update_cache: true


    - name: "Add Docker’s official GPG key:"
      shell: curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
      args:
        warn: no


    - name: "Add Docker’s official GPG key:"
      shell: sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
      args:
        warn: no


    - name: "Update the apt package index"
      apt:
        name: "*"
        state: latest


    - name: "Build-Step - Docker Installation"
      apt:
        pkg:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        state: present
        update_cache: true


    - name: "Build-Step - Additional package Installation"
      apt:
        pkg:
        - git
        - python3-pip
        state: present
        update_cache: true


    - name: "Build-Step - python docker extension installation"
      pip:
        name: docker-py


    - name: "Build-Step - Docker service restart/enable"
      service:
        name: docker
        state: restarted
        enabled: true


    - name: "Build-Step - Cloning Repository"
      git:
        repo: "{{ repo_url }}"
        dest: "{{ repo_dest }}"
      register: repo_status


    - name: "Build-Step - Login to remote Repository"
      when: repo_status.changed == true
      docker_login:
        username: "{{ docker_username }}"
        password: "{{ docker_password }}"


    - name: "Build-Step - Building image"
      docker_image:
        source: build
        build:
          path: "{{ repo_dest }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ repo_status.after }}"
        - latest


    - name: "Build-Step - removing image"
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ repo_status.after }}"
        - latest


- name: "Docker Run Container On Test Server"
  hosts: test
  become: true
  vars_files:
    - docker.vars

  tasks:

    - name: Update all packages to their latest version
      apt:
        name: "*"
        state: latest


    - name: "Install packages to allow apt to use a repository over HTTPS"
      apt:
        pkg:
        - apt-transport-https
        - ca-certificates
        - curl
        - gnupg
        - lsb-release
        state: present
        update_cache: true


    - name: "Add Docker’s official GPG key:"
      shell: curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
      args:
        warn: no


    - name: "Add Docker’s official GPG key:"
      shell: sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
      args:
        warn: no


    - name: "Update the apt package index"
      apt:
        name: "*"
        state: latest


    - name: "Build-Step - Docker Installation"
      apt:
        pkg:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        state: present
        update_cache: true


    - name: "Build-Step - Additional package Installation"
      apt:
        pkg:
        - git
        - python3-pip
        state: present
        update_cache: true


    - name: "Build-Step - python docker extension installation"
      pip:
        name: docker-py


    - name: "Build-Step - Docker service restart/enable"
      service:
        name: docker
        state: restarted
        enabled: true


    - name: "Deployment - Run Container"
      docker_container:
        name: webserver
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:5000"
~~~

## Set up the servers

1. CI/CD Server
2. Docker Image Build Server
3. Production/Test Server

We will be using ubuntu 20.04 Operating System on these three servers.

### 1. Set up the CI/CD Server

##### Run the following commands to install Jenkins
#
#
~~~
$ apt update
$ apt install openjdk-11-jdk
$ wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
$ sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
  /etc/apt/sources.list.d/jenkins.list'
$ apt-get update
$ apt-get install jenkins
~~~
![alt text](https://i.ibb.co/g31VWv7/jenkins-installation.png)

#### Run the following commands to install Ansible
~~~
$ apt update
$ apt install software-properties-common
$ add-apt-repository --yes --update ppa:ansible/ansible
$ apt install ansible
~~~

![alt text](https://i.ibb.co/9w0Zkgk/ansible-installation1.png)

### 2. Clone my Github repository to CI/CD Server

~~~
git clone https://github.com/sebinxavi/Docker-Container-Automation.git /var/playbook
~~~

![alt text](https://i.ibb.co/WngzcjV/git-clone1.png)

### 3. Update the ansible variables file and hosts file

##### hosts file (hosts)
#
~~~
[build]
172.31.37.78  ansible_user="ubuntu"

[test]
172.31.41.77  ansible_user="ubuntu"
~~~

- Update the IP address of the Build Server and Production/Test Server
- Update the user name of the operating system

##### Ansible variable file
#
~~~
repo_url: "https://github.com/sebinxavi/python-flask.git"
repo_dest: "/var/repository/"
image_name: "sebinxavi/flaskapp"
docker_username: "sebinxavi"
docker_password: ""
~~~
- Update the variables defined in the Ansible variable file (docker.vars)
- repo_url is the GitHub repository URL in which the application is running.

### 4. Install ansible plugin in Jenkins through "Manage Jenkins" option

![alt text](https://i.ibb.co/2ymmTJS/ansible-plugin-installation-jenkins.png)


### 5. Add ansible binary path through Jenkins Global Tool Configuration

![alt text](https://i.ibb.co/NKSLjnx/jenkins-add-ansible.png)


### 6. Create new Job in Jeniks by doing the following steps,

- Add the GitHub repository URL
- Add SSH private key and Ansible Vault Credentials
- Select GitHub hook trigger for GITScm polling
- Select Build as Ansible
- Add Playbook path as /var/playbook/main.yml
- Disable the host SSH key check
 
![alt text](https://i.ibb.co/GcWtBkg/job.png)

### 7. Check the console output through Jenkins

- Once the Job created, click on "Build now" and check the Console Output

![alt text](https://i.ibb.co/N7tckgq/console-output.png)

### 8. Add the Jenkins Payload URL in GitHUb repository's Webhook

![alt text](https://i.ibb.co/b6XyPhR/webhook-url.png)

9. You will able to see the changes once you updated the files inside the Application's GitHub repository.

-  Access the Production/Test Server's IP address in Browser
#
![alt text](https://i.ibb.co/ssFkXKw/latest-update.png)

## Author
Created by [@sebinxavi](https://www.linkedin.com/in/sebinxavi/) - feel free to contact me and advise as necessary!

<a href="mailto:sebin.xavi1@gmail.com"><img src="https://img.shields.io/badge/-sebin.xavi1@gmail.com-D14836?style=flat&logo=Gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/sebinxavi"><img src="https://img.shields.io/badge/-Linkedin-blue"/></a>
