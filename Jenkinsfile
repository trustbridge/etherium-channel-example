#!groovy

// Testing pipeline

pipeline {
    agent {
        label 'hamlet-latest'
    }
    options {
        timestamps ()
        buildDiscarder(
            logRotator(
                numToKeepStr: '10'
            )
        )
        disableConcurrentBuilds()
        durabilityHint('PERFORMANCE_OPTIMIZED')
        parallelsAlwaysFailFast()
        quietPeriod 60
    }

    environment {
        // Set poetry path location
        PATH = "$PATH:$HOME/.poetry/bin"
        DOCKER_BUILD_DIR = "${env.DOCKER_STAGE_DIR}/${BUILD_TAG}"
        COMPOSE_INTERACTIVE_NO_CLI = "1"
        COMPOSE_PROJECT_NAME = 'ethchn'
    }

    stages {

        stage('Setup') {
            steps {
                sh '''#!/bin/bash
                    curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
                    export PATH="$HOME/.poetry/bin:$PATH"
                    poetry --version
                '''
            }
        }

        stage('API Test') {
            steps {
                dir('channel-api/') {
                    sh '''#!/bin/bash
                        poetry install
                        poetry run pytest --junitxml=tests/junit.xml
                    '''
                }
            }

            post {
                always {
                    junit 'channel-api/tests/junit.xml'
                }
            }
        }

        stage('API Build') {
            steps {
                dir('channel-api/') {
                    sh '''#!/bin/bash
                    npm ci
                    export PATH="$(npm bin):$PATH"

                    sls package --package dist/
                    '''
                }
            }

            post {
                success {
                    dir('channel-api/') {
                        archiveArtifacts artifacts: 'dist/channel-api.zip', fingerprint: true
                    }
                }
            }
        }

        stage('Inbound Event listener') {

            steps {
                // Checkout into
                dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/") {
                    checkout scm
                }

                dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/inbound-event-listener") {
                    sh '''#!/bin/bash
                        docker-compose up -d --build --remove-orphans

                        echo "Having a nap while everything sets up"
                        sleep 30s

                        docker-compose exec -T worker flake8 --config=.flake8 src tests
                        docker-compose exec -T worker pytest --junitxml="/worker/test-report.xml"
                        docker-compose exec -T worker make coverage
                    '''
                }

            }

            post {
                always {
                    dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/inbound-event-listener") {
                        junit 'worker/test-report.xml'
                        publishHTML(
                            [
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'worker/htmlcov',
                                reportFiles: 'index.html',
                                reportName: 'Inbound Event Worker Coverage Report',
                                reportTitles: ''
                            ]
                        )
                    }
                }

                cleanup {
                    dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/inbound-event-listener") {
                        sh '''#!/bin/bash
                            docker-compose down --rmi local -v --remove-orphans
                        '''
                    }
                }
            }
        }

        stage('WebSub Hub') {

            steps {
                // Checkout into
                dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/") {
                    checkout scm
                }

                dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/websub-hub") {
                    sh '''#!/bin/bash
                        make build
                        make run

                        docker-compose exec -T websub-hub flake8 --config=.flake8 src tests
                        docker-compose exec -T websub-hub pytest --junitxml="/websub-hub/test-report.xml"
                        docker-compose exec -T websub-hub make coverage
                    '''
                }

            }

            post {
                always {
                    dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/websub-hub") {
                        junit 'worker/test-report.xml'
                        publishHTML(
                            [
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'websub-hub/htmlcov',
                                reportFiles: 'index.html',
                                reportName: 'Inbound Event Worker Coverage Report',
                                reportTitles: ''
                            ]
                        )
                    }
                }

                cleanup {
                    dir("${env.DOCKER_BUILD_DIR}/test/ethereum-channel/websub-hub") {
                        sh '''#!/bin/bash
                            docker-compose down --rmi local -v --remove-orphans
                        '''
                    }
                }
            }
        }
    }

    post {

        success {
            script {
                if ( env.BRANCH_NAME == 'master' ) {
                    build job: '../cotp-devnet/build-ethereum-channel/master', parameters: [
                        string(name: 'branchref_ethereum_channel', value: "${GIT_COMMIT}")
                    ]
                }
            }
        }

        cleanup {
            cleanWs()
        }
    }
}
