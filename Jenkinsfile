pipeline {
    agent any
    
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    
    
    environment {
        
        SCANNER_HOME= tool 'sonar-scanner'
    }
    // Gitlab Webhooks 
    // ##############
    triggers {
        GenericTrigger(
          genericVariables: [
            [key: 'ref', value: '$.ref'],
            [key: 'before', value: '$.before'],
            [key: 'after', value: '$.after'],
            [key: 'repo_url', value: '$.repository.url'],
          ],
          causeString: 'Triggered By Gitlab On $ref',
          token: 'git-lab-webhook',
          tokenCredentialId: '',
          // filter the deletion of release branch
          regexpFilterText: '$after',
          regexpFilterExpression: '^(?!0000000000000000000000000000000000000000$).*$',
          printContributedVariables: true,
          printPostContent: true,
          silentResponse: false,
        )
    }

    stages {
        stage('Git Checkout') {
            steps {
                git 'https://github.com/guneycansanli/secretsanta-generator.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Tests') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=santa \
                    -Dsonar.projectKey=santa -Dsonar.java.binaries=. '''
                }
            }
        }
        
        stage('Owasp Scan') {
            steps {
                dependencyCheck additionalArguments: ' --scan . ', odcInstallation: 'DC'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t santa:latest ."
                    }
                }
            }
        }
        
        stage('Tag & Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker tag santa:latest gnyscnsnli/santa:latest"
                        sh "docker push gnyscnsnli/santa:latest"
                    }
                }
            }
        }

        stage('Deploy Application') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker run -d -p 8081:8080 gnyscnsnli/santa:latest"
                    }
                }
            }
        }
        
    }
        // E-mail notification
         post {
            always {
                emailext (
                    subject: "Pipeline Status: ${BUILD_NUMBER}",
                    body: '''<html>
                                <body>
                                    <p>Build Status: ${BUILD_STATUS}</p>
                                    <p>Build Number: ${BUILD_NUMBER}</p>
                                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                                </body>
                            </html>''',
                    to: 'guneycansanli@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html'
                )
            }
        }
}
