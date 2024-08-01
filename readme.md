# DevSecOps Pipeline Setup - Final Project

## Authors
- Alan Kim
- Tao Sun
- Meedm Bossard
- Lihan Zhan
- Nicholas Mucks

## Software Used
- Docker 4.18.0 (104112)


## Prerequisites

- Ensure Docker and Docker Compose are installed on your machine. Follow the instructions on [Docker's official website](https://docs.docker.com/get-docker/) to install Docker and [Docker Compose installation](https://docs.docker.com/compose/install/).
- Docker Desktop can also be downloaded and installed on the system via https://www.docker.com/products/docker-desktop/


## Use Docker Compose to Set Up Containers for Jenkins, SonarQube, Prometheus, Grafana, and OWASP ZAP

1. **Create a Docker Compose file to define and run multi-container Docker applications**:
    - Use the provided `docker-compose.yml` in your project root directory. Here is the content of `docker-compose.yml`:
    ```yaml
    version: '3'
    services:
      jenkins:
        build: ./jenkins
        ports:
          - "8080:8080"
          - "50000:50000"
        volumes:
          - jenkins_home:/var/jenkins_home
      sonarqube:
        image: sonarqube
        ports:
          - "9000:9000"
      prometheus:
        image: prom/prometheus
        volumes:
          - ./prometheus.yml:/etc/prometheus/prometheus.yml
        ports:
          - "9090:9090"
      grafana:
        image: grafana/grafana
        ports:
          - "3000:3000"
      owasp-zap:
        image: owasp/zap2docker-stable
        ports:
          - "8081:8081"
    volumes:
      jenkins_home:
    ```

2. **Start the Docker Compose services**:
    - In the terminal, navigate to the directory containing your `docker-compose_spring_petclinic.yml` and run:
    ```bash
    docker-compose -f docker-compose_spring-petclinic.yml up -d
    ```
    - This Builds and Deploys docker containers using Docker Compose

## Fork the Project Repository on GitHub/GitLab and Clone It to Your Local Machine

1. **Navigate to the repository you want to fork on GitHub/GitLab**:
    - Go to the repository page on GitHub or GitLab.

2. **Click the "Fork" button in the top right corner**:
    - This will create a personal copy of the repository under your GitHub/GitLab account.

3. **Clone the forked repository to your local machine**:
    - Open a terminal and run the following commands:
    ```bash
    git clone <your-forked-repo-url>
    cd <your-repo-name>
    ```


## Create a Jenkins Pipeline That Uses the Forked GitHub Repository

1. **Open Jenkins in your web browser**:
    - Go to `http://localhost:8080`.

2. **Install the necessary plugins (GitHub, Pipeline)**:
    - Navigate to `Manage Jenkins > Manage Plugins`.
    - Install "GitHub Plugin" and "Pipeline Plugin".

3. **Create a new pipeline job in Jenkins**:
    - Go to `New Item`, enter a name, select `Pipeline`, and click `OK`.

4. **Configure the pipeline to use the Jenkinsfile from your forked repository**:
    - In the Pipeline section, select `Pipeline script from SCM`.
    - Choose `Git` and enter your repository URL.
    - Set the `Script Path` to `Jenkinsfile`.

## Set Up Build Triggers to Pull Source Control Management (SCM)

1. **In the Jenkins pipeline configuration, set up build triggers**:
    - Under `Build Triggers`, select `Poll SCM`.
    - Configure the polling schedule, for example:
    ```
    H/5 * * * *
    ```

## Create and Invoke Build Steps for the Spring-Petclinic Project

1. **Define the build steps in the Jenkinsfile to compile, test, and package the Spring-Petclinic project**:
    - Ensure your `Jenkinsfile` includes stages for building, testing, and packaging:
    ```groovy
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps {
                    sh 'mvn clean package'
                }
            }
            stage('Test') {
                steps {
                    sh 'mvn test'
                }
            }
            stage('Deploy') {
                steps {
                    // Deployment steps
                }
            }
        }
    }
    ```

## Configure SonarQube to Perform Static Analysis for the Project

1. **Install the SonarQube Scanner for Jenkins**:
    - Navigate to `Manage Jenkins > Manage Plugins`.
    - Install "SonarQube Scanner" plugin.

2. **Configure the SonarQube server details in Jenkins**:
    - Go to `Manage Jenkins > Configure System`.
    - Add SonarQube servers under `SonarQube Servers`.

3. **Add a SonarQube analysis stage to your Jenkinsfile**:
    ```groovy
    stage('SonarQube Analysis') {
        steps {
            withSonarQubeEnv('SonarQube') {
                sh 'mvn sonar:sonar'
            }
        }
    }
    ```

## Execute OWASP ZAP With Appropriate Configuration

1. **Add OWASP ZAP as a stage in your Jenkins pipeline**:
    - Use Docker to execute OWASP ZAP scan:
    ```groovy
    stage('OWASP ZAP Scan') {
        steps {
            sh 'docker run --rm -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-stable zap-baseline.py -t http://petclinic:8080'
        }
    }
    ```

## Add Post-Build Actions to Publish HTML Reports for OWASP ZAP

1. **In the Jenkins job configuration, add a post-build action to publish the HTML report**:
    - Go to `Post-build Actions` and select `Publish HTML reports`.
    - Specify the path to the ZAP report.

## Install the Prometheus Plugin in Jenkins

1. **Navigate to Manage Jenkins > Manage Plugins**:
    - Install the "Prometheus" plugin.

## Configure the Prometheus Plugin to Monitor Jenkins

1. **Configure the Prometheus plugin in Jenkins**:
    - Go to `Manage Jenkins > Configure System`.
    - Enable the "Expose metrics to Prometheus" option.

## Configure Grafana to Use Prometheus as a Data Source

1. **Open Grafana in your web browser**:
    - Go to `http://localhost:3000`.

2. **Add Prometheus as a data source in Grafana**:
    - Navigate to `Configuration > Data Sources > Add Data Source`.
    - Select `Prometheus` and enter the Prometheus URL (`http://prometheus:9090`).

3. **Create dashboards to visualize Jenkins metrics**:
    - Use the Grafana UI to create dashboards and add relevant panels to visualize metrics.

## Set Up a Virtual Machine (VM) to Act as the Production Web Server

1. **Create a VM using Vagrant or another virtualization tool**:
    - Install Vagrant from [Vagrant's official website](https://www.vagrantup.com/).
    - Create a `Vagrantfile` in your project directory:
    ```ruby
    Vagrant.configure("2") do |config|
      config.vm.box = "ubuntu/bionic64"
      config.vm.network "private_network", type: "dhcp"
      config.vm.provider "virtualbox" do |vb|
        vb.memory = "1024"
      end
    end
    ```
    - Start the VM:
    ```bash
    vagrant up
    ```

2. **Configure the VM to host the production version of the Spring-Petclinic application**:
    - SSH into the VM:
    ```bash
    vagrant ssh
    ```
    - Install necessary dependencies (Java, Maven, etc.) and deploy the Spring-Petclinic application.

## Use Ansible on the Jenkins Build Server to Deploy the Spring-Petclinic Application

1. **Create an Ansible playbook to deploy the application**:
    - Create a file named `deploy.yml`:
    ```yaml
    - hosts: web
      tasks:
        - name: Deploy Spring-Petclinic
          copy:
            src: /path/to/your/application
            dest: /path/on/the/production/server
    ```

2. **Add an Ansible deployment stage to your Jenkinsfile**:
    ```groovy
    stage('Deploy to Production') {
        steps {
            sh 'ansible-playbook -i inventory deploy.yml'
        }
    }
    ```

## Ensure the Application is Deployed and Running on the Production Web Server

1. **Verify the application is running by accessing it via the web browser**:
    - Open your web browser and navigate to the VM's IP address or hostname.

2. **Check the Jenkins build logs for any deployment errors**:
    - Ensure there are no errors in the Jenkins build logs.

## Make and Push a Code Change to the GitHub Repository

1. **Make a code change to your local repository**:
    - Edit a file in your repository.

2. **Commit and push the changes**:
    ```bash
    git add .
    git commit -m "Made a code change"
    git push origin main
    ```

## Verify Jenkins Automatically Builds, Tests, and Deploys the New Version

1. **Monitor the Jenkins pipeline to ensure it triggers automatically**:
    - Check the Jenkins dashboard to see if the pipeline starts automatically after pushing the code changes.

2. **Verify the build, test, and deployment steps complete successfully**:
    - Ensure all stages in the Jenkins pipeline complete without errors.

## Deliverables

1. **Step-by-Step Instructions (Grading - 30 points)**: Detailed documentation on setting up and configuring the DevSecOps pipeline.
2. **Pipeline Setup and Configuration Files (Grading - 30 points)**: Submit the Docker commands and scripts used, including Dockerfiles, Vagrant files, Groovy scripts, YAML files, etc.

---

This document provides a comprehensive guide to set up and configure the DevSecOps pipeline for your project. Follow each step carefully to ensure a successful deployment.
