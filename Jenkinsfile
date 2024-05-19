pipeline {
    agent { label 'Jenkins-Agent' }
    tools {
        jdk 'java17'
        maven 'maven3'
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
    }
}
