#### Reg_App_CI-CD_Pipeline -The Following Command Lines are step by Step instructions to create this pipeline.

      #============================================================= Bootstrap Eksctl, K8 and ArgoCD setup=============================================================
      #create a eks-bootstrap server with t2.micro,8gb ram and ubuntu 22.0 flavor #change hostname 'EKS'if needed
      #create and assign ec2 -adminaccess and attach to the device
      #Run the Following as a script to save time
      sudo apt update
      sudo apt upgrade -y
      curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
      sudo apt install unzip -y
      unzip awscliv2.zip
      sudo ./aws/install
      curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl
      sudo chmod +x ./kubectl
      sudo mv kubectl /bin
      curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
      sudo mv /tmp/eksctl /bin
      aws --version
      kubectl version --output=yaml
      eksctl version
      
      #Cluster creation
      eksctl create cluster --name eks-cluster --region ap-south-1 --node-type t2.small --nodes 3
      kubectl get nodes
      
      #ArgoCD Installation
      kubectl create namespace argocd
      kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
      kubectl get pods -n argocd #verification
      #ArgoCD CLI installation
      curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
      chmod +x /usr/local/bin/argocd
      kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
      kubectl get svc -n argocd
      kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
      echo MFNJOW0xNGo4T2R5SndwMQ== | base64 --decode
      argocd login a20f74cff370646a6979556aba2c4247-474294886.ap-south-1.elb.amazonaws.com --username admin
      alias k=kubectl
      k config get-contexts
      argocd cluster add i-015e2724bd5703fe1@eks-cluster.ap-south-1.eksctl.io  --name eksctl-cluster
      argocd cluster list
      kubectl config get-contexts
      kubectl get svc
      #Login using the URL and change the password
      
      #============================================================= Install and Configure Jenkins-Master & Jenkins-Agent=============================================================
      #Create 3 VM's in EC2 with ubuntu 22.04,t2.medium and 20GB Named: Jenkins-Master,Jenkins-Agent,SonarQube
      #Rename all three devices if needed by sudo vi /etc/hostname and reboot the devices by sudo init 6
      
      ## create a script and chmod +x and run it to save time
      ## Install Java and Jenkins (on both servers) #refer official documentaion for commands (on both servers) https://www.jenkins.io/doc/book/installing/linux/ 
      
      sudo apt update
      sudo apt upgrade -y
      sudo apt install openjdk-17-jre -y
      
      sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
        https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
      echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
        https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
        /etc/apt/sources.list.d/jenkins.list > /dev/null
      sudo apt-get update
      sudo apt-get install jenkins -y
      sudo systemctl enable jenkins      
      sudo systemctl start jenkins       
      systemctl status jenkins #verify
      java -version #verify
      
      ##Install docker (only in Jenkins-agent server)
      sudo apt-get install docker.io -y
      sudo usermod -aG docker $USER
      
      #To connect Master and Agent perform these on Both Servers
      sudo vi /etc/ssh/sshd_config
      publickeyauthentication yes #change 
      AuthorizedkeysFile #uncomment
      sudo service sshd reload
      
      #Establishing ssh connection from jenkins-master to server. Note: #run the following as ubuntu user 
      #create a ssh-keygen in Jenkins-Master, get the pub key located at .ssh/
      ssh-keygen -t ed25519 or ssh-keygen #uses ed25519 algorithm instead of defalut Rivest-Shamir-Adleman AKA RSA algorithm #more secure more fast
      cat /home/ubuntu/.ssh/id_ed25519.pub
      #copy these contents to the authorized key files in 'Jenkins-agent' server which is located at '/home/ubuntu/.ssh/authorized_keys' authorizedkey
      #paste it in a new line
      sudo vi /home/ubuntu/.ssh/authorized_keys
      
      #open jenkins-server in the URL with port 8080 and configure the basic details
      sudo cat /var/lib/jenkins/secrets/initialAdminPassword #for getting password
      
      
      %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
      ##Adding jenkins-agent to master. In jenkins management console and installing and adding plugins
        1. Manage Jenkins-> Nodes -> Built in Node -> Configure ;Number of executors=0 and save #to avoid using master node for building Jobs
        2. Manage Jenkins -> Nodes -> Create New -> Name=Jenkins-Agent;Permenant = True; Number of executors=2;Remote Root Directory=/home/ubuntu;Labels=Jenkins-Agent; 
              Usage=use this node as much as possible; Launch Method=launch agents via ssh; Host=private ip of agent;  
              credentials->add ;kind=ssh username with privatekey;ID=Jenkins-Agent-Creds;Username=ubuntu;Private key='get the private key from jenkins-master under cat /home/ubuntu/.ssh/id_ed25519' #public key is already added on jenkins-agent 
              Hostkeyverification=non verifing strategy
        3. Create a test pipeline with hello world job and verify its running successfully on jenkins-agent and delete the job after testing
        4. #installing plugins
           Manage Jenkins-> plugins -> Available plugins = 'Maven integration & Pipline Maven Integration & eclipse temurin installer' and install 
        5. #configuring java17
           Manage Jenkins-> tools -> JDK installations -> name=java17 ;  Install automatically add installer - install from adoptium.net -> jdk 17.0.5+8 
           #configuing maven 
           and get to  maven installations -> name=maven3;Install automatically ; apply and save 
        6. #adding github creds to jenkins
            Manage Jenkins-> credentials -> username=githubusername; password= personal access token from github site; ID= github-creds token : github-token-value
        7. #Docker plugin installation and configuration
           Manage Jenkins -> plugins -> available plugins -> Docker,Docker Commons,Docker API,Docker pipeline,Docker-build-step,cloudbees docker build and push #restart jenkins after install
           Manage jenkins -> Credentials -> Global -> Add Creds- -> username - docker username ; token: docker-tonen-value
           #Note without restarting jenkins server docker login will fail 
      
      %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
      #============================================================= Install and configure Postgres, Sonarqube and Trivy=============================================================
      #Perform these on the SonarQube Server
      #Postgres Installation. creating a script for the below to save time also works
      sudo apt update
      sudo apt upgrade -y
      sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
      wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
      sudo apt update
      sudo apt-get -y install postgresql postgresql-contrib
      sudo systemctl enable postgresql
      
      #creating database for sonarqube
      sudo passwd postgres
      su - postgres
      createuser sonar #This command creates a new PostgreSQL user named sonar. The createuser command is a PostgreSQL utility for adding new roles (users).
      psql  #opens PostgreSQL interactive terminal
          ALTER USER sonar WITH ENCRYPTED password 'sonar'; #This SQL command sets the password for the sonar PostgreSQL user to sonar, and stores the password in an encrypted format.
          CREATE DATABASE sonarqube OWNER sonar; #This SQL command creates a new PostgreSQL database named sonarqube and assigns the sonar user as the owner of this database.
          grant all privileges on DATABASE sonarqube to sonar; #This SQL command grants all privileges on the sonarqube database to the sonar user
          \q
      exit
      
      ## Add Adoptium repository and install java
      sudo bash
      wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
      echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
      apt update
      apt install temurin-17-jdk -y
      update-alternatives --config java
      /usr/bin/java --version
      exit 
      
      ## Linux Kernel Tuning
      # Increase Limits
      sudo vi /etc/security/limits.conf #    Paste the values at the bottom of the file
          sonarqube   -   nofile   65536
          sonarqube   -   nproc    4096
      # Increase Mapped Memory Regions
      sudo vi /etc/sysctl.conf # Paste the below values at the bottom of the file
          vm.max_map_count = 262144
      sudo init 6
      
      #### Sonarqube Installation ####
      ## Download and Extract
      sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
      sudo apt install unzip -y
      sudo unzip sonarqube-9.9.0.65466.zip -d /opt
      sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube
      ## Create user and set permissions
      sudo groupadd sonar
      sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar
      sudo chown sonar:sonar /opt/sonarqube -R
      ## Update Sonarqube properties with DB credentials
      sudo vi /opt/sonarqube/conf/sonar.properties #edit the following values and create it if needed
           sonar.jdbc.username=sonar
           sonar.jdbc.password=sonar
           sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
      ## Create service for Sonarqube
      sudo vi /etc/systemd/system/sonar.service #Paste the follwing
          [Unit]
          Description=SonarQube service
          After=syslog.target network.target
      
          [Service]
          Type=forking
      
          ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
          ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
      
          User=sonar
          Group=sonar
          Restart=always
      
          LimitNOFILE=65536
          LimitNPROC=4096
      
          [Install]
          WantedBy=multi-user.target
      ## Start Sonarqube and Enable service
      sudo systemctl start sonar
      sudo systemctl enable sonar
      sudo systemctl status sonar
      ## Watch log files and monitor for startup
      sudo tail -f /opt/sonarqube/logs/sonar.log
      
      #sonarqube works on port-9000 login and change the password
      #From sonarqube console create a token for jenkins Administrator->My account->security ->genrate Token #sample Token- sonarqube-token
      
      %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
      #Add the sonarqube token in Jenkins
      Manage jenkins -> Credentials -> Global -> Add Creds- -> secret text ;Id=jenkins-sonarqube-token
      #Install sonarqube plugins 
      Manage Jenkins -> plugins -> available plugins -> SonarQube Scanner+Sonar Quality Gates + Quality Gates and Intall and restart jenkins when completed  sudo systemctl restart jenkins
      #configuring sonarqube server to jenkins
      Manage Jenkins -> system -> Sonarqube server -> Sonarqubeinstallation -> serverURL=http://172.31.32.18:9000 ;token=jenkins-sonarqube-token(#private ip of sonarqube server)
      #adding sonarqube scanner to the tools
      Manage jenkins -> tools-> SonarQube Scanner installations -> sonarqube-scanner;Install Automatically
      
      #Create webhook in sonarqube
      #in sonarqube console
      Administration -> configuration -> webhooks -> create Name=sonarqube-webhook ;URL=http://172.31.42.43:8080/sonarqube-webhook/ #privateip of jenkins-master server
      
      #Now create a pipleline job in Jenkins called - Register-CI
      1. Choose Discard Old build ->Max build to Keep=2 #to save Space
      2. Pipeline script from SCM -> Add repo URL of Github which we will create ;Add github-creds ;Choose the main branch and the jenkinsfile and save 
      #Now add this to github Jenkinsfile and modify the names of creds as needed
      pipeline {
          agent { label 'Jenkins-Agent' }
          tools {
              jdk 'java17'
              maven 'maven3'
          }
          environment {
      	    APP_NAME = "register-app"
                  RELEASE = "1.0.0"
                  DOCKER_USER = "velkaandhan"
                  DOCKER_PASS = 'docker-creds'
                  IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
                  IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"	    
          }
          stages{
              stage("Cleanup Workspace"){
                      steps {
                      cleanWs()
                      }
              }
      
              stage("Checkout from SCM"){
                      steps {
                          git branch: 'main', credentialsId: 'github-creds', url: 'https://github.com/Velkaandhans/Reg_App_CI.git'
                      }
              }
      
              stage("Build Application"){
                  steps {
                      sh "mvn clean package"
                  }
      
             }
      
             stage("Test Application"){
                 steps {
                       sh "mvn test"
                 }
             }
             stage("SonarQube Analysis"){
                 steps {
                       script {
                           withSonarQubeEnv(credentialsID: 'jenkins-sonarqube-token') {
                           sh "mvn sonar:sonar"
                           }
                       }
                  }
             }
            stage("Quality Gate"){
                steps {
                    script {
                          waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                      }	
                  }
              }
           stage("Build & Push Docker Image") {
                  steps {
                      script {
                          docker.withRegistry('',DOCKER_PASS) {
                              docker_image = docker.build "${IMAGE_NAME}"
                          }
                          docker.withRegistry('',DOCKER_PASS) {
                              docker_image.push("${IMAGE_TAG}")
                              docker_image.push('latest')
                          }
                      }
                  }
              }
             stage("Trivy Scan") {
                 steps {
                     script {
      	            sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image velkaandhan/register-app:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
                     }
                 }
             }
      
             stage ('Cleanup Artifacts') {
                 steps {
                     script {
                          sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                          sh "docker rmi ${IMAGE_NAME}:latest"
                     }
                }
             }
          }
      }
      #also upload the necessary source binaries in the repo and trigger the job from Jenkins #the files can be taken from - https://github.com/Velkaandhans/Reg_App_CI.git
      @@@@@@@@@@#The CI part of the Job is Done.
      
      
      #ArgoCD and Github integration
      #Create a new repo in github consisting of the manifest files #refer https://github.com/Velkaandhans/Reg_App_CI_Manifest-files.git for manifest files
      Settings->Repositories->Connect Repo -> Enter URL;githubusername;and the github Token
      
      #In you github repo modify dep.yaml -> image name and in Jenkinsfile modify url's
      #create a new app in ArgoCD
      SyncPolicy:automatic;pruneresource;selfheal;autocreatenamespace;replace ; repourl: githuburl;path:./;destination:eksctl-cluster;namespace:default
      #Test the newly created URL with port 8080/webapp
      @@@@@@@@@@#The CD part of the Job is Done.
      
      #============================================================= Integrating CI and CD part =============================================================
      #In Jenkins Managemet Console
      #Create a new CD job as a pipeline
      Discard old builds; max builds to keep -2; This project is parameterized -string parameter - IMAGE_TAG;Trigger builds remotely;gitops-token;pipelinescript from SCM; githubURL ;main ; githubcreds ;scriptpath: Jenkinsfile
      #create a new jenkins token for authentication 
      admin->configure->API token->JENKINS_API_TOKEN: token-value
      #add this token to the credential manger
      Manage Jenkins-> Credentials-> add credentials->secret text-> ID:JENKINS_API_TOKEN ;Description:JENKINS_API_TOKEN
      #Modify the CI repo now to include the trigger for CD Job and add the new variable #add the public dns URL for jenkins-master
      #In this stage we send the git-ops  token and Imagetag value to the CD job to trigger
      stage("Trigger CD Pipeline") {
                  steps {
                      script {
                          sh "curl -v -k --user admin:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-15-206-210-102.ap-south-1.compute.amazonaws.com:8080/job/Reg-App-CD/buildWithParameters?token=gitops-token'"
                      }
                  }
             }
      
      #The CICD Pipeline is done and The new changes will auto trigger the CI and CD Job which will reflect in the URL.

#### After Completing
#### delete the eks cluster
    eksctl delete cluster --region=ap-south-1 --name=eks-cluster
#### Delete all the VM's

