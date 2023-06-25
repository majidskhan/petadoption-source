pipeline {
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('nexus-repo')
    }
    stages {
        stage('Code Analysis') {
            steps {
               withSonarQubeEnv('sonarqube') {
                  sh "mvn sonar:sonar"  
               }
            }   
        }
        stage("Quality Gate") {
            steps {
              timeout(time: 2, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
        stage('Build Artifact') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/myapp:latest .'
            }
        }
        stage('Log into Nexus Repo') {
            steps {
               sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO' 
            }
        }
        stage('Push to Nexus Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/myapp:latest'
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent (['ansible-key']) {
                      sh 'ssh -t -t ec2-user@35.180.91.255 -o StrictHostKeyChecking=no "cd /etc/ansible && ansible-playbook stage-trigger.yml -e ansible_python_interpreter=/usr/bin/python3"'
                }
            }
        }
        stage('slack notification') {
            steps {
                slackSend channel: 'Cloudhight',
                message: 'App deployed to Stage, needs approval to deploy to prod',
                teamDomain: '29th-may--pet-adoption-containerisation-ansible-auto-discovery--eu-team-2',
                tokenCredentialId: 'slack'
            }
        }
        stage('Request for Approval'){
            steps{
                timeout(activity: true, time: 5){ 
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        } 
        stage('Deploy to prod'){
            steps{
                sshagent(['ansible-key']) {
                  sh 'ssh -t -t ec2-user@35.180.91.255 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook prod-trigger.yml -e ansible_python_interpreter=/usr/bin/python3"'
                }
            }
        } 
    }                   
}
