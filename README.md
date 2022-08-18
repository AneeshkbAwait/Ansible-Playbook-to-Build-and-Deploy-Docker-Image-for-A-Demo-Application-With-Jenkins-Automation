## Ansible-Playbook-to-Build-and-Deploy-Docker-Image-for-A-Demo-Application
This is a Playbook to automate the build and deployment of a Docker-based application. The application code is placed in the GitHub Repository. We will fetch the application code from the GitHub repo and then build a Docker image.

The entire workflow is as follows:

1. The developer team will put the application code in a GitHub Repository.

2. Using the Ansible Playbook, we will build the application Docker image using the code placed in the GitHub repository, and then we will push the image to Dockerhub with the tag "latest".

3. Later, we will build a Docker container using the image tagged as' latest 'fetched from Dockerhub.

4. Jenkins is used in this case because we want to automate the build and deploy process in response to an application change or version update. This means, whenever the development team implements any code change/maintenance or application version update, a new Docker image will be constructed by taking the changes applied and it will be pushed to the Dockerhub. So, whenever a new Docker image is pushed to the Dockerhub, the Docker container will be recreated with the new Docker image so as to run the latest application code. The Jenkins GitHub Webhook is used to trigger the action whenever developers commit something into the repository.

### Ansible Inventory:
Docker image is built and pushed to dockerhub using the host 'docker-builder'.
Application container is created using these images on host 'app-server'
```
[docker-builder]
172.31.43.114 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="key.pem"

[app-server]
172.31.37.69 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="key.pem"
```

### credential.vars
credential.vars is a variable file which contains login credential to Dockerhub.
```
---
dock_user: "testacct"
dock_pwd: "***********"
```

### Playbook for build and deploying docker image:
```
---
######## Play to Build Docker Image & to Push it to DockerHub ######## 
- name: "Installing Docker and Building an Image Git Cloned Docker File"
  hosts: docker-builder
  become: true
  vars_files:
    - credential.vars
  vars:
    git_url: https://github.com/AneeshkbAwait/devops-flask.git
    git_clone_dir: /home/ec2-user/docker_cloned_images
    packages:
      - git
      - docker
      - python-pip
  tasks:


    - name: "Installing docker & Git packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker-py python module"
      pip:
        name: docker-py

    - name: "Adding Ec2-user to the docker group"
      user:
        name: ec2-user
        groups: docker
        append: yes

    - name: "Starting and enabling Services"
      service:
        name: docker
        state: restarted
        enabled: true

    - name: "Cloning Dockerfile from Git respository"
      git:
        repo: "{{ git_url }}"
        dest: "{{ git_clone_dir }}"
      register: clone_status

    - name: "Build the image from the git cloned Dockerfile"
      when: clone_status.changed == true
      docker_image:
        build:
          nocache: yes
          rm: yes
          path: "{{ git_clone_dir }}"
        name: "{{ dock_user }}"/ansible-flaskapp
        tag: "{{ clone_status.after }}"
        source: build
      register: docker_build

    - name: 'Retag Current Built Image as Latest'
      when: docker_build.changed == true
      shell: docker image tag "{{ dock_user }}"/ansible-flaskapp:"{{ clone_status.after }}" "{{ dock_user }}"/ansible-flaskapp:latest

    - name: "Log into DockerHub"
      when: clone_status.changed == true
      docker_login:
        username: "{{ dock_user }}"
        password: "{{ dock_pwd }}"
        state: present
      register: login_status

    - name: "Pushing the Image to Docker Hub"
      when: clone_status.changed == true and login_status.changed == true
      docker_image:
        name: "{{ dock_user }}/ansible-flaskapp:{{ item }}"
        push: yes
        force_tag: yes
        source: local
      with_items:
        - latest
        - "{{ clone_status.after }}"
      register: push_status

    - name: "Log out from  DockerHub"
      docker_login:
        state: absent

    - name: "Removing the Local Images once the Push is Success!!"
      when: docker_build.changed == true and push_status.changed == true
      docker_image:
        name: "{{ dock_user }}/ansible-flaskapp:{{ item }}"
        state: absent
      with_items:
        - "{{ clone_status.after }}"
        - latest

######## Play to Deploying the Docker Image in app-server ######## 


- name: "Installing Docker and Creating Container from ansible-flaskapp Image"
  hosts: app-server
  become: true
  vars:
    packages:
      - docker
      - python-pip
  tasks:


    - name: "Installing docker and python-pip packages"
      yum:
        name: "{{ packages }}"
        state: present
      register: instal_status

    - name: "Installing docker-py python module"
      pip:
        name: docker-py

    - name: "Adding Ec2-user to the docker group"
      user:
        name: ec2-user
        groups: docker
        append: yes

    - name: "Starting and enabling Services"
      when: instal_status.changed == true
      service:
        name: docker
        state: restarted
        enabled: true

    - name: "Pulling the Docker Image from Docker Hub"
      docker_image:
        name: aneeshkbawaits/ansible-flaskapp:latest
        source: pull
        force_source: yes
        state: present
      register: pull_status

    - name: "Creating docker container"
      when: pull_status.changed == true
      docker_container:
        name: flaskapp
        image: aneeshkbawaits/ansible-flaskapp:latest
        state: started
        restart_policy: always
        recreate: yes
        published_ports: 80:5000
        detach: yes
```

Here the Playbook has 2 plays; 

The First play will build docker images based on the application dockerfile available at GitHub and then pushes it to the DockerHub. Whenever there is a change in the application version/any code updation, this play will build and push docker images automatically by applying the changes.

The Second play will always fetch the latest docker Image of the application and will recreate the application containers whenever a new docker image is pushed to the DockerHub.

### Jenkins:

The Ansible Playbook is executed through a jenkins job which created for the purpose.

![alt text](https://s3.ap-south-1.amazonaws.com/githubpjts.aneeshponnu.tech/PB+job+created+Step+34.JPG "Jenkins Job for Playbook execution")

Jenkins GitHub Webhook is used to trigger the build and deployment process whenever Developers commit something into the repository. So, whenever there is an updated application code is placed in the GitHub repo, the image build and deploy automatically carried out. 

![alt text](https://s3.ap-south-1.amazonaws.com/githubpjts.aneeshponnu.tech/GitHub+hook+trigger+enabled+Step+43.JPG "Jenkins Job for Playbook execution")
