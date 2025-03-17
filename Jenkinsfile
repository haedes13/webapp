pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/haedes13/webapp.git'
            }
        }

        stage('Initialize') {
            steps {
                echo "PATH = ${env.PATH}"
                echo "M2_HOME = ${env.M2_HOME}"
            }
        }

        stage('Check-Git-Secrets') { 
            steps { 
                sh 'rm -f trufflehog || true' // Remove previous scan results if they exist
                sh 'docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog'
                sh 'cat trufflehog' // Display scan results
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -X' // Added -X for debug output
            }
        }

        stage('Deploy-To-Tomcat') {
            steps {
                sshagent(['tomcat']) { 
                    sh 'scp -o StrictHostKeyChecking=no target/*.war tomcat@192.168.59.177:/home/tomcat/apache-tomcat-9.0.102/webapps/webapp.war'
                }
            }
        }
    }

    post {
        success {
            echo 'Build and Deployment succeeded!'
        }
        failure {
            echo 'Build or Deployment failed!'
        }
    }
}
