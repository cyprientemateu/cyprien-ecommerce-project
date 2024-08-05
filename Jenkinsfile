pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '3'))
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        timestamps()
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: '')
        string(name: 'SONAR_VERSION', defaultValue: '5.0.1', description: '')
    }

    environment {
        DOCKER_CREDENTIALS_ID = 'tcc-docker-hub'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project") {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.BRANCH_NAME}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [[$class: 'LocalBranch']],
                        submoduleCfg: [],
                        userRemoteConfigs: [[
                            url: 'https://github.com/cyprientemateu/cyprien-ecommerce-project.git',
                            credentialsId: 'tcc-github-access'
                        ]]
                    ])
                }
            }
        }

        stage('Scan Golang Code') {
            agent {
                docker {
                    image 'golang:1.22.5'
                    args '-u root' // Run the container as the root user
                }
            }
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project/do-it-yourself/src/catalog") {
                    sh '''
                    ls -la                                                 
                    uname -r
                    go test
                    '''
                }
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

        stage('Test') {
            steps {
                sh '''
                ls
                pwd
                whoami
                '''
            }
        }

        stage('SonarQube analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:5.0.1'
                }
            }
            environment {
                CI = 'true'
                scannerHome = '/opt/sonar-scanner'
            }
            steps {
                withSonarQubeEnv('Sonar') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'tcc-docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USER --password-stdin'
                }
            }
        }

        stage('Docker build ui') {
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project/do-it-yourself/src/ui") {
                    sh '''
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-ui:${BUILD_NUMBER} .
                    '''
                }
            }
        }

        stage('Docker build catalog') {
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project/do-it-yourself/src/catalog") {
                    sh '''
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-catalog:${BUILD_NUMBER} .
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-catalog-db:${BUILD_NUMBER} . -f Dockerfile-db
                    '''
                }
            }
        }

        stage('Docker build checkout') {
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project/do-it-yourself/src/checkout") {
                    sh '''
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-checkout:${BUILD_NUMBER} .
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-checkout-db:${BUILD_NUMBER} . -f Dockerfile-db
                    '''
                }
            }
        }

        stage('Docker build orders') {
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project/do-it-yourself/src/orders") {
                    sh '''
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-orders:${BUILD_NUMBER} .
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-orders-db:${BUILD_NUMBER} . -f Dockerfile-db
                    '''
                }
            }
        }

        stage('Docker build cart') {
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project/do-it-yourself/src/cart") {
                    sh '''
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-cart:${BUILD_NUMBER} .
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-cart-dynamo-db:${BUILD_NUMBER} . -f Dockerfile-dynamodb
                    '''
                }
            }
        }

        stage('Docker build assets') {
            steps {
                dir("${WORKSPACE}/cyprien-ecommerce-project/do-it-yourself/src/assets") {
                    sh '''
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-assets:${BUILD_NUMBER} .
                    docker build -t cyprientemateu/a1cyprien-do-it-yourself-assets-rabbitmq:${BUILD_NUMBER} . -f Dockerfile-rabbitmq
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend color: '#2EB67D',
            channel: 'development-alerts',
            message: "*a1cyprien-do-it-yourself Project Build Status*" +
            "\n Project Name: do-it-yourself" +
            "\n Job Name: ${env.JOB_NAME}" +
            "\n Build number: ${currentBuild.displayName}" +
            "\n Build Status : *SUCCESS*" +
            "\n Build url : ${env.BUILD_URL}"
        }
        failure {
            slackSend color: '#E01E5A',
            channel: 'development-alerts',
            message: "*a1cyprien-do-it-yourself Project Build Status*" +
            "\n Project Name: do-it-yourself" +
            "\n Job Name: ${env.JOB_NAME}" +
            "\n Build number: ${currentBuild.displayName}" +
            "\n Build Status : *FAILED*" +
            "\n Action : Please check the console output to fix this job IMMEDIATELY" +
            "\n Build url : ${env.BUILD_URL}"
        }
        unstable {
            slackSend color: '#ECB22E',
            channel: 'development-alerts',
            message: "*a1cyprien-do-it-yourself Project Build Status*" +
            "\n Project Name: do-it-yourself" +
            "\n Job Name: ${env.JOB_NAME}" +
            "\n Build number: ${currentBuild.displayName}" +
            "\n Build Status : *UNSTABLE*" +
            "\n Action : Please check the console output to fix this job IMMEDIATELY" +
            "\n Build url : ${env.BUILD_URL}"
        }
    }
}