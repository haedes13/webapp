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

        stage('Build') {
            steps {
                sh 'mvn clean package -X' // Added -X for debug output
            }
        }

        stage('Verify SSH Key') {
            steps {
                sshagent(['tomcat']) {
                    // Test SSH connection to the remote server
                    sh '''
                        echo "Testing SSH connection to tomcat@192.168.59.177..."
                        ssh -o StrictHostKeyChecking=no -v tomcat@192.168.59.177 "echo SSH connection successful!"
                    '''
                }
            }
        }

        stage('Deploy-To-Tomcat') {
            steps {
                sshagent(['tomcat']) {
                    // Copy the WAR file to the remote server
                    sh '''
                        echo "Deploying WebApp.war to tomcat@192.168.59.177..."
                        scp -o StrictHostKeyChecking=no -v target/*.war tomcat@192.168.59.177:/tmp/prod/apache-tomcat-9.0.102/webapps/webapp.war
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sshagent(['tomcat']) {
                    // Verify the file was copied successfully
                    sh '''
                        echo "Verifying deployment on tomcat@192.168.59.177..."
                        ssh -o StrictHostKeyChecking=no tomcat@192.168.59.177 "ls -l /tmp/prod/apache-tomcat-9.0.102/webapps/webapp.war"
                    '''
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
