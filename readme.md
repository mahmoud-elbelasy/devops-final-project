# DevOps Final Project: CI/CD Pipeline for a Python Web Application

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [Configure Ansible](#configure-ansible)
5. [Usage](#usage)
6. [Project Structure](#project-structure)
7. [Future Improvements](#future-improvements)

---

## Overview

This project demonstrates the implementation of a CI/CD pipeline to build, deploy, and run a Python-based web application using Jenkins, Docker, Vagrant, and Ansible. The pipeline automates the following tasks:

- Pulling the application code from a GitHub private repository.
- Building and pushing a Docker image to Docker Hub.
- Provisioning two Vagrant virtual machines as target nodes.
- Installing Docker on the target nodes using Ansible.
- Pulling the Docker image from Docker Hub and running the application container on both target nodes.

The CI/CD pipeline ensures streamlined and efficient deployment of the application, reducing manual intervention and improving reliability. For more details, see [CI/CD Pipeline Details](#ci-cd-pipeline-details).

---
![IMG000](https://github.com/user-attachments/assets/60dc72ee-7441-4a78-a344-2f1beaad4afd)
![Screenshot 2024-12-21 132211](https://github.com/user-attachments/assets/1f8dd26b-6330-47a1-b2c0-e7b254362ce3)


## Prerequisites

Before running this project, ensure you have the following installed:

- **Git**: Version control system.
- **Docker**: For containerization.
- **Vagrant**: To manage virtual machines.
- **Ansible**: For configuration management.
- **Jenkins**: For CI/CD pipeline.

---

## Project Setup

### 1. Clone the Repository
Clone the project repository from GitHub:
```bash
git clone (https://github.com/mahmoud-elbelasy/devops-final-project/tree/main)
```

### 2. Dockerize the Application
The application is containerized using the following Dockerfile:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY . /app

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

CMD ["python", "app.py"]
```

Build and push the Docker image:
```bash
docker build -t mahmoudbelasy/web-app:latest .
docker push mahmoudbelasy/web-app:latest
```

### 3. Vagrant Virtual Machines
Create two Vagrant machines using the Vagrantfile.
```ini
Vagrant.configure("2") do |config|
  # Base box configuration
  base_box = "eurolinux-vagrant/centos-stream-9" # You can change this to another lightweight box

  ### VM 1 ###
  config.vm.define "vm1" do |vm1|
    vm1.vm.box = base_box
    vm1.vm.hostname = "vm1"
    vm1.vm.network "private_network", ip: "192.168.56.101" # Static IP for VM 1
    vm1.vm.provider "virtualbox" do |vb|
      vb.memory = "600" # Minimal memory
      vb.cpus = 1       # Minimal CPU
    end
  end

  ### VM 2 ###
  config.vm.define "vm2" do |vm2|
    vm2.vm.box = base_box
    vm2.vm.hostname = "vm2"
    vm2.vm.network "private_network", ip: "192.168.56.102" # Static IP for VM 2
    vm2.vm.provider "virtualbox" do |vb|
      vb.memory = "600" # Minimal memory
      vb.cpus = 1       # Minimal CPU
    end
  end
end
```

### 4. Configure Ansible
Ansible inventory file (`inventory.ini`):
```ini
[vms]
192.168.56.101 ansible_user=vagrant ansible_private_key_file=keys/vm1_key
192.168.56.102 ansible_user=vagrant ansible_private_key_file=keys/vm2_key
```

### 5. Ansible Playbook
Create a playbook (`deploy.yml`) to:
- Install Docker on the target machines.
- Pull the Docker image from Docker Hub.
- Run the container on the target machines.
```yaml
---
- name: update machine, install docker, pull image and run image
  hosts: vms
  become: yes
  tasks:

    - name: Install prerequisites
      yum:
       name:
          - yum-utils
          - device-mapper-persistent-data
          - lvm2
       state: present

    # Check if Docker is already installed
    - name: Check if Docker is installed
      command: docker --version
      register: docker_installed
      failed_when: false
      changed_when: false

    # Add Docker repository (only if Docker is not installed)
    - name: Add Docker repository
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
      when: docker_installed.rc != 0

    # Install Docker (only if Docker is not installed)
    - name: Install Docker
      yum:
        name: docker-ce
        state: present
      when: docker_installed.rc != 0

    # Start and enable Docker service
    - name: Ensure Docker is running
      service:
        name: docker
        state: started
        enabled: yes

    # Pull a Docker image
    - name: Pull Docker image
      docker_image:
        name: mahmoudbelasy/pipeline-devops-final-project:latest
        source: pull

    # Run the container on port 5000
    - name: Run the container
      docker_container:
        name: flask_app
        image: mahmoudbelasy/pipeline-devops-final-project:latest
        state: started
        ports:
          - "5000:5000"
```
---

## CI/CD Pipeline Details

### Jenkins Pipeline

The Jenkinsfile contains the following stages:
1. **Clone Repository**: Pull the code from the GitHub repository.
2. **Build Docker Image**: Build the Docker image using the provided Dockerfile.
3. **Push Docker Image**: Push the image to Docker Hub.
4. **Run Ansible Playbook**:
    - Install Docker on the Vagrant nodes.
    - Pull and run the Docker container.

Example Jenkins pipeline code:
```groovy
pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-cred') 
        DOCKER_HUB_REPO = 'mahmoudbelasy/pipeline-devops-final-project'
        IMAGE_TAG = "latest"
    }
    stages {
        stage('pull code') {
            steps {
                git branch: 'main', credentialsId: 'github-cred', url: 'https://github.com/mahmoud-elbelasy/devops-final-project.git'
            }
        }
        stage('build docker image') {
            steps {
                sh 'docker build -t ${DOCKER_HUB_REPO}:${IMAGE_TAG} .'
               
            }
        }
        stage('push docker image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                        docker logout
                    '''
                }
            }
        }
        stage('run ansible playbook and run the container') {
            steps {
                sh 'chmod 600 keys/vm1_key'
                sh 'chmod 600 keys/vm2_key'
                
                sh 'ansible-playbook -i inventory.ini playbook_test.yml'
            }
        }
    }
    
    post {
        always {
            echo "Cleaning up unused Docker images..."
            sh 'docker system prune -f'
        }
    }
}
```

![photo_2024-12-24_06-58-36](https://github.com/user-attachments/assets/77c59905-a87d-4720-8531-c5edc0629b59)

---

## Usage

1. **Start Jenkins**: Run Jenkins on your local machine or server.
2. **Create a Jenkins Job**:
   - Use the provided `Jenkinsfile` as the pipeline script.
   - Add necessary credentials for GitHub and Docker Hub.
3. **Run the Pipeline**: Trigger the pipeline to automate the application deployment.

---

## Project Structure

```sh
└── devops-final-project/
    ├── app.py
    ├── Dockerfile
    ├── inventory.ini
    ├── jenkins-file.txt
    ├── keys
    │   ├── vm1_key
    │   └── vm2_key
    ├── playbook_test.yml
    ├── requirements.txt
    ├── static
    │   └── .gitkeep
    └── templates
        ├── index.html
        └── weather.html
```

---

## Future Improvements

- Implement health checks for the deployed containers.
- Automate the creation of Vagrant nodes using a script.
- Set up monitoring tools like Prometheus and Grafana.
- Add rollback mechanisms in case of deployment failure.

---
