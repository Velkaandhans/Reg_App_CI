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

