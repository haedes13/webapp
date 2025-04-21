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
                sh 'rm -f trufflehog || true'
                sh 'docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog'
                sh 'cat trufflehog'
            }
        }

        stage('Source Composition Analysis') {
            steps {
                sh 'rm owasp* || true'
                sh 'wget "https://raw.githubusercontent.com/haedes13/webapp/refs/heads/master/owasp-dependency-check.sh"'
                sh 'chmod +x owasp-dependency-check.sh'
                sh 'bash owasp-dependency-check.sh'
                sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
            }
        }

        stage('SAST') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn sonar:sonar'
                    sh 'cat target/sonar/report-task.txt'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -X'
            }
        }

        stage('Deploy-To-Tomcat') {
            steps {
                sshagent(['tomcat']) { 
                    sh 'scp -o StrictHostKeyChecking=no target/*.war tomcat@192.168.59.177:/home/tomcat/apache-tomcat-9.0.102/webapps/webapp.war'
                }
            }
        }

        stage('Port Scanning') {
            steps {
                sh '''
                    echo "Running Nmap port scan on Tomcat server..."
                    nmap -sT -T4 -p- 192.168.59.177 -oN portscan.txt

                    echo "Formatting port scan output:"
                    grep '^PORT' -A 100 portscan.txt | awk '/open/{print $1, $2, $3}' > formatted-ports.txt
                    cat formatted-ports.txt

                    echo "Checking for unexpected open ports..."
                    UNEXPECTED=$(awk '{print $1}' formatted-ports.txt | cut -d/ -f1 | grep -Ev '^(22|80|8080)$' || true)

                    if [ ! -z "$UNEXPECTED" ]; then
                      echo "❌ Unexpected open ports detected:"
                      echo "$UNEXPECTED"
                      exit 1
                    else
                      echo "✅ Only expected ports are open."
                    fi
                '''
            }
        }

        stage('DAST') {
            steps {
                sshagent(['zap']) {
                    sh '''
                    ssh -o StrictHostKeyChecking=no owaspzap@192.168.59.180 '
                      docker run -v /tmp:/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                      -t http://192.168.59.177:8080/webapp/ \
                      -r zap-report.html \
                      -J zap-report.json \
                      -x zap-report.xml
                    '
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'portscan.txt, formatted-ports.txt', onlyIfSuccessful: false
        }
        success {
            echo '✅ Build and Deployment succeeded!'
        }
        failure {
            echo '❌ Build or Deployment failed!'
        }
    }
}
