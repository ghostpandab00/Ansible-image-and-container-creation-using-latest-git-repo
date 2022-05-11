# Ansible-image-and-container-creation-using-latest-git-repo

## Description
We'll go over how to make a docker image and container from the most recent git repo in this repository. I'm creating the image on a build server and the container on a test server.

## Ansible Modules used

  1. yum
  2. pip
  3. service
  4. git
  5. docker_image
  6. docker_container
  7. docker_login

## The code's backstory
```sh
[build]

18.118.2.87 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="key.pem"

[test]

3.145.12.43 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="key.pem"
```

```sh
---
- name: "Building Docker Image in build server"
  hosts: build
  become: true
  vars:
    packages:
      - git
      - pip
      - docker
    repo_url: "https://github.com/vyshnavlal/git-flask-app.git"
    repo_dir: "/home/ec2-user/flaskapp/"
    docker_user: "<docker_user>"
    docker_password: "<docker_password>"
    image_name: "vyshnavlal/flaskrep"

  tasks:
    - name: " installing packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker client for python"
      pip:
        name: docker-py

    - name: "adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting/enabling Docker"
      service:
        name: docker
        state: started
        enabled: true

    - name: "Clonning the repo using {{ repo_url }}"
      git:
        repo: "{{repo_url}}"
        dest: "{{ repo_dir }}"
      register: git_status


    - name: "Logging into the docker-hub official"
      when: git_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"


    - name: "Creating docker Image and push to hub"
      when: git_status.changed == true
      docker_image:
        source: build
        build:
          path: "{{ repo_dir }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ git_status.after }}"
        - latest

    - name: "Logout from hub"
      when: git_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: absent

    - name: "Removing docker image from build server"
      when: git_status.changed == true
      docker_image:

        name: "{{ image_name }}"
        tag: "{{ item }}"
        state: absent
      with_items:
        - "{{ git_status.after }}"
        - latest


- name: "creating container in test server"
  hosts: test
  become: true
  vars:
    packages:
      - pip
      - docker
    image_name: "vyshnavlal/flaskrep"
  tasks:
    - name: " installing packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker client for python"
      pip:
        name: docker-py

    - name: "adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting/enabling Docker"
      service:
        name: docker
        state: started
        enabled: true


    - name: "Pulling the docker Image from hub "
      docker_image:
        name: "{{image_name}}:latest"
        source: pull
        force_source: true
      register: image_status

    - name: " Creating the Container from pulled image"
      when: image_status.changed == true
      docker_container:
        name: flaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:5000"
```
We could see that a container was built on the client server after running the playbook.

```sh
#docker container ls

CONTAINER ID   IMAGE                     COMMAND            CREATED         STATUS         PORTS                  NAMES
c68a5ecf4510   vyshnavlal/flaskrep:latest   "python3 app.py"   2 minutes ago   Up 2 minutes   0.0.0.0:80->5000/tcp   flaskapp
```

It will skip the creation and build if we perform the same without making any changes to the docker image repository.

## Conclusion
We spoke about how to make a docker image and container from the newest git repo in this lesson. If the developer makes any changes to the dockerfile that is stored in git. This will completely recreate the image and container without skipping.
