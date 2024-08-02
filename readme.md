# DevSecOps Pipeline Setup - Final Project

## Authors
- Alan Kim
- Tao Sun
- Meedm Bossard
- Lihan Zhan
- Nicholas Mucks

## Software Used
- docker 4.18.0 (104112)
- grafana v10.2.8
- sonarqube:8.9-community
- prometheus v2.35
- Spring Framework v6.1.2
- OWASP ZAP Proxy v3.0.3
- AWS EC2 (Instance Type: t2.micro)
- ansible v403.v8d0ca_dcb_b_502
- Java v22

 
## Jenkins Plugins Used
- ansible v403.v8d0ca_dcb_b_502
- docker-workflow:580.vc0c340686b_54 
- prometheus v2.35
- configuration-as-code 
- blueocean v1.27.14
- sonarqube:8.9-community
- json-path-api:2.8.0-5.v07cb_a_1ca_738c 
- token-macro:400.v35420b_922dcb_ 
- favorite:2.218.vd60382506538 
- github:1.39.0 
- github-branch-source:1790.v5a_7859812c8d


## Prerequisites

- Ensure Docker and Docker Compose are installed on your machine. Follow the instructions on [Docker's official website](https://docs.docker.com/get-docker/) to install Docker and [Docker Compose installation](https://docs.docker.com/compose/install/).
- Docker Desktop can also be downloaded and installed on the system via https://www.docker.com/products/docker-desktop/


## Fork the Project Repository on GitHub/GitLab and Clone It to Your Local Machine

1. **Navigate to the repository you want to fork on GitHub/GitLab**:
    - Go to the repository page on GitHub or GitLab.

2. **Click the "Fork" button in the top right corner**:
    - This will create a personal copy of the repository under your GitHub/GitLab account.

3. **Clone the forked repository to your local machine**:
    - Open a terminal and run the following commands:
    ```bash
    git clone https://github.com/spring-projects/spring-petclinic
    cd <your-repo-name>
    ```


## Use Docker Compose to Set Up Containers for Jenkins, SonarQube, Prometheus, Grafana, and OWASP ZAP

1. **Create a Docker Compose file to define and run multi-container Docker applications**:
    - Here is the content of `docker-compose_spring-petclinic.yml`:
    ```
    version: '3'

    services:
      jenkins:
        build:
          context: .
          dockerfile: Dockerfile.jenkins
        ports:
          - "8081:8080"
          - "50000:50000"
        networks:
          - custom-network
        privileged: true
        user: root
        volumes:
          - ./jenkins_data:/var/jenkins_home
          - ./jenkins.yaml:/var/jenkins_home/casc_configs/jenkins.yaml
          - /var/run/docker.sock:/var/run/docker.sock
        environment:
          - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
          - CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs/jenkins.yaml
        env_file:
          - .env


      prometheus:
        image: prom/prometheus:latest
        ports:
          - "9090:9090"
        networks:
          - custom-network
        volumes:
          - ./prometheus_data:/prometheus
          - ./prometheus.yml:/etc/prometheus/prometheus.yml

      grafana:
        image: grafana/grafana:latest
        ports:
          - "3000:3000"
        networks:
          - custom-network
        volumes:
          - ./grafana_data:/var/lib/grafana
          - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
          - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards
        environment:
          - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD}
          - GF_SECURITY_ALLOW_EMBEDDING=true
          - GF_AUTH_DISABLE_LOGIN_FORM=true
          - GF_AUTH_ANONYMOUS_ENABLED=true
          - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin

        depends_on:
          - prometheus
        env_file:
          - .env

      db:
        image: postgres:12
        networks:
          - custom-network
        volumes:
          - ./postgresql:/var/lib/postgresql
          - ./postgresql_data:/var/lib/postgresql/data
        environment:
          POSTGRES_USER: sonar
          POSTGRES_PASSWORD: sonar

      sonarqube:
        image: sonarqube:community
        ports:
          - "9000:9000"
          - "9092:9092"
        networks:
          - custom-network
        volumes:
          - ./sonarqube_conf:/opt/sonarqube/conf
          - ./sonarqube_data:/opt/sonarqube/data
          - ./sonarqube_logs:/opt/sonarqube/logs
          - ./sonarqube_extensions:/opt/sonarqube/extensions
          - ./sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins
        environment:
          SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
          SONAR_JDBC_USERNAME: sonar
          SONAR_JDBC_PASSWORD: sonar
        depends_on:
          - db
        env_file:
          - .env

    volumes:
      prometheus_data:
      grafana_data:
      jenkins_data:
      sonarqube_conf:
      sonarqube_data:
      sonarqube_logs:
      sonarqube_extensions:
      sonarqube_bundled-plugins:
      postgresql:
      postgresql_data:

    networks:
      custom-network:
    ```

2. **Start the Docker Compose services**:
    - In the terminal, navigate to the directory containing your `docker-compose_spring_petclinic.yml` and run:
    ```bash
    docker-compose -f docker-compose_spring-petclinic.yml up -d
    ```
    - This Builds and Deploys docker containers using Docker Compose


## Create a Jenkins Pipeline That Uses the Forked GitHub Repository

1. **Open Jenkins in your web browser**:
    - Go to `http://localhost:8081`.

2. **Install the necessary plugins (GitHub, Pipeline)**:
    - Navigate to `Manage Jenkins > Manage Plugins`.
    - Install "GitHub Plugin" and "Pipeline Plugin".

3. **Create a new pipeline job in Jenkins**:
    - Go to `New Item`, enter a name
    - Select `Pipeline`, 
    - Use `Git` and specify your repository URL.
    - Set the `Script Path` to `Jenkinsfile`.
    - click `OK`. 


## Set Up Build Triggers to Pull Source Control Management (SCM)

1. **In the Jenkins pipeline configuration, set up build triggers**:
- Start with `Configure` on the left hand side on our Jenkins pipeline
- Click on `Scan Multi-branch pipeline triggers` 
- Check `Periodically if not otherwise run`
- For interval, select `1 min` or `2 min` based on your preference


## Create and Invoke Build Steps for the Spring-Petclinic Project

1. **Define the build steps in the Jenkinsfile to compile, test, and package the Spring-Petclinic project**:
    - Our pipeline is set up to pull JenkinsFile from our Github repository automatically and used it for our Jenkin's groovy file
    - Jenkinsfile content:
    ```
    pipeline {
        agent any

        environment {
            SONARQUBE_URL = 'http://sonarqube:9000'
            SONARQUBE_CREDENTIALS_ID = 'admin'
            GITHUB_TOKEN = credentials('github-token')
        }

        stages {
            stage('Checkout') {
                steps {
                    script {
                        echo "Checking out code..."
                        git url: 'https://github.com/CChariot/spring-petclinic.git', branch: 'FinalProject_main', credentialsId: 'github-token'
                    }
                }
            }

            stage('Build Docker Image') {
                steps {
                    script {
                        echo "Building Docker Image..."
                        def dockerImage = docker.build("spring-petclinic")
                        echo "Docker Image built: ${dockerImage.id}"
                        // Store the Docker image ID in the environment if needed across stages
                        env.DOCKER_IMAGE_ID = dockerImage.id
                    }
                }
            }

            stage('Deploy Container') {
                steps {
                    ansiblePlaybook(
                        playbook: 'deploy.yml',
                        inventory: 'inventory'
                    )
                }
            }        

        }

        post {
            always {
                script {
                    // Use the saved Docker image ID from the environment if needed
                    if (env.DOCKER_IMAGE_ID) {
                        echo "Stopping and removing Docker Image with ID: ${env.DOCKER_IMAGE_ID}"
                        docker.rmi(env.DOCKER_IMAGE_ID)
                    }
                }
            }
            success {
                echo 'Pipeline completed successfully!'
            }
            failure {
                echo 'Pipeline failed.'
            }
        }
    }
    ```

## Configure SonarQube to Perform Static Analysis for the Project

1. **Set up SonarQube server**
    - Go to sonarqube server http://localhost:9000
    - Login with following credentials and reset password when prompted:
        username: admin
        password: admin
    - Click on “Create a local project”
    - Create a project with following specifications:
        Project display name: petclinic
        Project key: petclinic
        Main branch name: main
    - Choose “Use the global setting” then create the project
    - For the Analysis Method, choose “Locally”
    - To Analyze your project, choose Generate a project token, use the following specifications, then generate token:
        Token name: Analyze “petclinic”
        Expires in: 30 days
    - Copy this token

2. **Create credential SonarQube server and SonarQube scanner on Jenkins**
    - Go to jenkins server http://localhost:8081
    - Navigate to: Manage Jenkins->Configurations
    - Choose global, then add credentials
    - Create new credential with following fields:
        Kind: Secret text
        Scope: Global
        Secret: Paste token from Sonarqube server
        ID: sonarqube_token
    
    

    **Setting up plugins for SonarQube server**
    - Navigate to: Manage Jenkins->Systems->SonarQube servers
    - Set the following fields:
        Name: test_sonarqube
        Server URL: http://sonarqube:9000
        Server authentication token: Choose sonarqube_token

        
    **Setting up plugins for SonarQube scanner**
    - Navigate to: Manage Jenkins->Tools
    - Set following fields for SonarQube Scanner installations:
        Name: sonar-scanner
        Version: SonarQube Scanner 4.7.0.2747



3. **Add a SonarQube analysis stage to your Jenkinsfile**:
    ```
    stage('SonarQube analysis') {
            steps {

                script {
                    scannerHome = tool 'sonar-scanner'// must match the name of an actual scanner installation directory on your Jenkins build agent
                }
                withSonarQubeEnv('test_sonarqube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dproject.settings=sonar-project.properties
                        """
                }

            }
        }
    ```

    **SonarQube Property file content**:
    ```
    sonar.projectKey=petclinic
    sonar.projectName=petclinic
    sonar.projectVersion=1.0
    sonar.sources=.
    sonar.exclusions=**/*.java
    sonar.token=$SONAR_TOKEN
    ```

    **SonarQube DockerCompose file content**:
    ```
    db:
        image: postgres:12
        networks:
          - custom-network
        volumes:
          - ./postgresql:/var/lib/postgresql
          - ./postgresql_data:/var/lib/postgresql/data
        environment:
          POSTGRES_USER: sonar
          POSTGRES_PASSWORD: sonar

      sonarqube:
        image: sonarqube:community
        ports:
          - "9000:9000"
          - "9092:9092"
        networks:
          - custom-network
        volumes:
          - ./sonarqube_conf:/opt/sonarqube/conf
          - ./sonarqube_data:/opt/sonarqube/data
          - ./sonarqube_logs:/opt/sonarqube/logs
          - ./sonarqube_extensions:/opt/sonarqube/extensions
          - ./sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins
        environment:
          SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
          SONAR_JDBC_USERNAME: sonar
          SONAR_JDBC_PASSWORD: sonar
        depends_on:
          - db
        env_file:
          - .env
    ```



## Execute OWASP ZAP With Appropriate Configuration
    
   1.  Build the Docker container for the OWASP ZAP
   2.  Within the container, run the ZAP baseline pipeline script against our petclinic VM
   3.  Copy the output to our Volume
   4.  Once succeeded, a report will be generated


   **Add OWASP ZAP as a stage in your Jenkins pipeline**:
    
    ```
    stage('Run ZAP Scan') {
            steps {
                script {
                    // Create the zap-report directory
                    sh "mkdir -p ${WORKSPACE}/zap-report"

                    // Pull the ZAP image
                    def zapImage = docker.image("ghcr.io/zaproxy/zaproxy:stable")

                    // Run the ZAP scan
                    try {
                    // -I to ignore errors
                    // -j to include inline css
                        zapImage.inside("-v ${WORKSPACE}/zap-report:/zap/wrk:rw --name zap-scan --rm") {
                            sh "zap-baseline.py -t http://3.149.247.7:8080 -g gen.conf -I -j -r zap-report.html"
                            // Ensure the report is copied to the mounted volume
                            sh "cp /zap/wrk/zap-report.html ${WORKSPACE}/zap-report/"
                        }

                        // Add a small delay before container cleanup
                        sleep 5

                        // Debug steps
                        sh "ls -la ${WORKSPACE}/zap-report"
                        sh "cat ${WORKSPACE}/zap-report/zap-report.html || echo 'ZAP report not found'"
                    } catch (Exception e) {
                        echo "Warnings found during ZAP scan: ${e.message}"
                    }
                }
            }
        }
    }
    ```

## Add Post-Build Actions to Publish HTML Reports for OWASP ZAP

1. **In the Jenkins job configuration, add a post-build action to publish the HTML report**:
    - Save the ZAP report as a page within Build page of Jenkin called `OWASP ZAP report` 
    - Click `OWASP ZAP Report` to view the report
    
    Jenkins file content:
    ```
    post {
            always {
                publishHTML(target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: false,
                    keepAll: true,
                    reportDir: "${WORKSPACE}/zap-report",
                    reportFiles: 'zap-report.html',
                    reportName: 'OWASP_ZAP_Report'
                ])

                 // Archive the ZAP report as a downloadable artifact
                archiveArtifacts artifacts: 'zap-report/*.html', fingerprint: true
            }
            success {
                echo 'Pipeline completed successfully!'
            }
            failure {
                echo 'Pipeline failed.'
            }
        }
    }

    ```

## Install the Prometheus Plugin in Jenkins

1.  Configure `prometheus.yml` and add two scrape configs - 1 for Jenkins & 1 for Prometheus

    Scrape Configs Content:
    ```
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'jenkins'
        metrics_path: '/prometheus/'
        static_configs:
          - targets: ['jenkins:8080']

      - job_name: 'prometheus'
        static_configs:
          - targets: ['prometheus:9090']
    ```

2.  The Dockerfile to build Jenkin includes a step to install the Prometheus plugin
3.  Confirm that Docker Compose is running
  
    Prometheus Docker Compose content: 
    ```
    prometheus:
      image: prom/prometheus:latest
      ports:
        - "9090:9090"
      networks:
        - custom-network
      volumes:
        - ./prometheus_data:/prometheus
        - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ```


## Configure the Prometheus Plugin to Monitor Jenkins

1. **Configure the Prometheus plugin in Jenkins**:
    - Go to `Manage Jenkins > Configure System`.
    - Enable the "Expose metrics to Prometheus" option.


## View the Prometheus data
1. Go to `http://localhost:8081/prometheus` to view the log data from Jenkins
2. Go to Prometheus (`http://localhost:9090/targets`) to view the jobs


## Configure Grafana to Use Prometheus as a Data Source

1. **Open Grafana in your web browser**:
    - Go to `http://localhost:3000`.

2. **Add Prometheus as a data source in Grafana**:
    - Navigate to `Configuration > Data Sources > Add Data Source`.
    - Select `Prometheus` 
    - Enter the Prometheus URL (`http://prometheus:9090`).

3. **Create dashboards to visualize Jenkins metrics**:
    - Use the Grafana UI to create dashboards and add relevant panels to visualize metrics.
    - At the bottom of the page, select `Metrics` and click on the type of metrics you'd like to visualize
    - Click `Apply` to view the metrics


## Set Up a Virtual Machine (VM) to Act as the Production Web Server

1. **Create a VM using EC2 Instance**:
    - Register and login using your AWS Account
    - Once inside, navigate to EC2 dashboard
    - Click on `Instances`
    - Click `Launch Instances` on top right corner
    - Give it a name - Ex: Ansible
    - Select `Amazon Linux 2023 AMI version`
    - Inside of Architecture, select `64-bit (x86)`
    - Within the firewall setting, click `Create Security Group` and `Allow SSH traffic from anywhere`
    - Click on `Launch Instance` on bottom right corner
    - Wait for the instance to be running. Once running, click on the instance to find the Public IPv4 DNS
    - Configure inventory file for ansible to include latest EC2 IP address 
      
      Inventory file Content: 
      ```
      [webservers]
      ec2-3-149-247-7.us-east-2.compute.amazonaws.com ansible_user=ec2-user ansible_ssh_private_key_file=/var/jenkins_home/ansible.pem
      ```

    - Creating a Key Pair - Click on the Key Pair on the left-hand tab in EC2 home
    - Click on `Create Key Pair` on the top-right-hand side
    - Give it a name - Ex: Ansible
    - For the Key Pair Type: `RSA option` 
    - For the Private Key file format, select `.pem`
    - Click on `Create Key Pair`
    - Copy the content of the Key Pair into the ansible.pem file

    Content:
    ```
    -----BEGIN RSA PRIVATE KEY-----
    MIIEpAIBAAKCAQEAuk3Fpp+uD3H7LbywPoN5iH5xBrQZkYv3xOYmPAkIQewes0Hr
    WFXJrpCFppuB0duJEZVyE8hB6PFogYEy5KNE6W56DKwl3CFHmC0MyrnwlIH+LD2w
    9H5JkGTe6SVHpMsvJtQgcTdMKz331GJueFTIP2HIj5qOF2zAAfuBEY2x9Sc4YIJx
    R5kXz6imiRfjZEFrFo0ioQe63qRVepCb2RKz+/bzOgkn0+0CVGcXrbgemgEN954O
    pVwIhv/0yV/E980dW+q+PiG1ZySE3WLe+cSJyzztkQA+h7uHpfBoumLo3Sw3EPxT
    qL3D8v1eoWqzCEmBPvx7TT2On/gm9cmudRhHXwIDAQABAoIBAAqHirw4GiZVUtTq
    7SsbUysbuleepjNLrd07BL4v5H+VUMbg2uRLNPLgyCz6bQPnXH/Z6nCjyNXZjwaC
    vtWdRK/MxqkgsaMXXmyDX0215JsAHdVyRyYKXS4EBXU33iy6LxgKtSqw7WUkQ3WF
    eqjiYc7zP9qd6Zn5U4DJLipHz98DF7ZCNPQfxXDhAqccFCIG3pGPh0/KzZWPPjRG
    /J83sBf4ejuV/gq9HiFaw+uuN1S3qWOPF1z3xSrYJVWr5cbs0Y8R86gna8AZf1Jc
    Sr5Sb7G5TezUlJ6Y4m4JaaXd5e/cvdP5zOoC8yYg+vF4ZzIVS+WutumL8KXkhNqV
    owlHEkECgYEA6c/7+KvUM1DQOdUSeeOx9Jcq0F7iXltl9w0X4L0zQOakavinO/PX
    VgSMpuJYWKYD4S/CHq3h1Wl04xpbLmvXTkWH/j9y41IulwJaDBrhqGaesxrKX7P6
    ZRW5JoTE6DGXbP9KPiS/O6ZeQu0+37Y2XK9bxyitFdR0oF+nh80g7RsCgYEAy/uo
    h6cNxO8xwLz0mbye50M/bWXwdnmCq5ev9deQ+61VsoyQQPlM55RaWjt35LavWD0L
    SUIuTXmDUN3uO6xnH4tcerweyu1XSERkQUXsasgS6viGfvP+/Zew7AEfHzxCkIQs
    xRRQj2mkAqc5DKbp0E/cp2hgN2ybqgj3va1Zhw0CgYAw29x8n3ONcaLBowvkWrdy
    NDCnMFy/ePv6v0qxFPhj5I6RJ/rSZWcnO3Yk3YG2rKJ86Rz4ij95+DqLxpMtRS3N
    1mvPrnSUmjTQK5ajlu524VLifIOzsgluHDb/nJkFKG/LQCHEkKtBjMd/1tHfr9T2
    U1KrcI2S1T210adRkoUB5wKBgQCdKwpvewfg9WwgVXch/XNyPR5h7Gma34UPMZEi
    mzXatXOSXzvG1E+tH2F+pNN8JkZ0dpR7ncKPb1D+vgEReYT7iSV4a/pN4RGfXRLi
    OD4xCHeLFHKM3vNZ8ccgEL0qFAQ11aGpOD3aQktcv/v1A6akGuSpGIMKMWS/XqmE
    PEz/AQKBgQDnkigevSCzXi43/mQBVzjAlxmXeg/SN5GVCRmPTsxz1qWQSY1aLe+I
    t2a5lz62ETX3xvfWhCh7kCdI886HB00UTIgP/2Vqwpysb0cGDWxkfpNjtsmZ36wN
    gH1k+UNCIIYizw4Y0LTrKxYW7RPXfZ80IlqFcvVsssTAoUoKYl6f2w==
    -----END RSA PRIVATE KEY-----
    ```

    Note: This will serve as the SSH key for logging into the VM in our Jenkins pipeline

    - Within EC2, click the instance that was just created
    - Select `Security` tab 
    - Click on Security Group for Inbound rules (Ex: launch-wizard-1)
    - Once inside of Security Group, click on Security Group ID
    - Click on Edit Inbound Rules
    - For the type, select `Custom TCP`
    - For the port range, select `8080`
      Note: This is our petclinic port
    - Within the source option, select `0.0.0.0/0`
    - Click `Save Rules`


## Use Ansible on the Jenkins Build Server to Deploy the Spring-Petclinic Application

1. **Create an Ansible playbook to pull Docker image and start Docker service of petclinic that we built in our Jenkins pipeline**
    
    Content:
    ```
    - hosts: webservers
      become: yes
      tasks:
        - name: Ensure Python 3 is installed
          package:
            name: python3
            state: present

        - name: Install Docker on Debian-based systems
          when: ansible_os_family == "Debian"
          apt:
            name: docker.io
            state: present
            update_cache: yes

        - name: Install Docker on RedHat-based systems
          when: ansible_os_family == "RedHat"
          yum:
            name: docker
            state: present

        - name: Start Docker service
          service:
            name: docker
            state: started
            enabled: yes

        - name: Pull Docker image
          docker_image:
            name: chariot97/petclinic
            tag: latest
            source: pull

        - name: Run Docker container
          docker_container:
            name: spring-petclinic
            image: chariot97/petclinic:latest
            state: started
            ports:
              - "8080:8080"

    ```

2. **Add an Ansible deployment stage to your Jenkinsfile**:
    
    Content:
    ```
    stage('Add SSH Key and Deploy Container to VM') {
            steps {
                script {
                    // Add the SSH key to known hosts and set permissions
                    sh '''
                        mkdir -p ~/.ssh
                        chmod 700 ~/.ssh
                        ssh-keyscan -H ec2-3-149-247-7.us-east-2.compute.amazonaws.com >> ~/.ssh/known_hosts
                        chmod 600 ~/.ssh/known_hosts
                        chmod 600 /var/jenkins_home/ansible.pem
                    '''
                    // Execute the Ansible playbook
                    ansiblePlaybook(
                        playbook: '/var/jenkins_home/deploy.yml',
                        inventory: '/var/jenkins_home/inventory',
                        extras: '--private-key=/var/jenkins_home/ansible.pem'
                    )
                }
            }
        }
    ```

3. **Wait for the instance to be running. Once running, click on the EC2 instance that was created within the previous EC2 set-up to find the Public IP:**

    - Configure inventory file for ansible to include latest EC2 IP address 
      
      Inventory file Content: 
      ```
      [webservers]
      ec2-3-149-247-7.us-east-2.compute.amazonaws.com ansible_user=ec2-user ansible_ssh_private_key_file=/var/jenkins_home/ansible.pem
      ```
    

## Ensure the Application is Deployed and Running on the Production Web Server

1. **Trigger the deployment through our Jenkins pipeline**:
    - Take the public IP of EC2 instance and add port 8080 to see if application is accessible from the an external environment - Ex: http://3.149.247.7:8080/
    

2. **Verify the ansible stage within our Jenkins pipeline to confirm successful build**:
    - Check the Jenkins build logs for any deployment errors


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




This document provides a comprehensive guide to set up and configure the DevSecOps pipeline for your project. Follow each step carefully to ensure a successful deployment.
