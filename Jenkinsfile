pipeline {
    agent{
        any
    }
     
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        timestamps()
    }
    
    triggers {
        githubPush()
    }

    stages {
        stage('Scan Golang Code') {
            agent {
                docker {
                    image 'golang:1.22.5'
                    args '-u root' // Run the container as the root user
                }
            }
            steps {
                sh '''
                cd do-it-yourself/src/catalog/
                ls -la                                                 
                uname -r
                go test
                '''
            }
        }
        
        stage('Unit Test UI Code') {
            agent {
                docker {
                    image 'maven:3.8.5-openjdk-17'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock' // Mount Docker socket
                }
            }
            environment {
                DOCKER_HOST = 'unix:///var/run/docker.sock' // Set Docker host
            }
            steps {
                sh '''
                cd do-it-yourself/src/ui
                mvn test
                '''
            }
        }

        stage('Unit Test Cart Code') {
            agent {
                docker {
                    image 'maven:3.8.5-openjdk-17'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock' // Mount Docker socket
                }
            }
            
            steps {
                sh '''
                cd do-it-yourself/src/cart
                mvn test -DskipTests=true
                '''
            }
        }

        stage('Unit Test Orders Code') {
            agent {
                docker {
                    image 'maven:3.8.5-openjdk-17'
                    args '-u root' // Run the container as the root user
                }
            }
            steps {
                sh '''
                cd do-it-yourself/src/orders
                mvn test -DskipTests=true
                '''
            }
        }
        
        stage('Scan Checkout Code') {
            agent {
                docker {
                    image 'node:14' // Use a Node.js Docker image
                    args '-u root' // Run the container as the root user
                }
            }
            steps {
                sh '''
                cd do-it-yourself/src/checkout
                ls -la
                node --version
                npm --version
                # Add other scanning or commands as needed
                '''
            }
        }

        // stage('SonarQube Scan') {
        //     // agent any // Run on any available agent

        //     steps {
        //         script {
        //             def scannerHome = tool name: 'SonarQubeScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
        //             withSonarQubeEnv('SonarQube Server') {
        //                 sh "${scannerHome}/bin/sonar-scanner"
        //             }
        //         }
        //     }
        // }

        stage('Test') {
            steps {
                sh '''
                ls
                pwd
                whoami
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}