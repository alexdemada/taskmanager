pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'alexjrc972/taskshare'
        DOCKER_CREDENTIALS = 'credential-dockerhub'
        SONARQUBE_ENV = 'Sonarqube'
        SONAR_HOST_URL = 'http://192.168.27.28:9000/'
        KUBECONFIG_PATH = '/var/lib/jenkins/.kube/config'
        // Note : On ne met pas SONARQUBE_TOKEN ici car on l'injecte via withCredentials
    }

    stages {
        stage('Cloner le code') {
            steps {
                git branch: 'main', url: 'https://github.com/alexdemada/taskmanager.git'
            }
        }

        stage('Tests & couverture') {
            steps {
                dir('backend') {
                    sh 'npm install'
                    sh 'npx nyc --reporter=lcov --reporter=text mocha ./test/basic.test.js --reporter mocha-junit-reporter --reporter-options mochaFile=./test-results/results.xml'
                }
            }
        }

        stage('Analyse SonarQube') {
            steps {
                withCredentials([string(credentialsId: 'credential-sonarqube', variable: 'SONARQUBE_TOKEN')]) {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        sh """
                            sonar-scanner \
                            -Dsonar.projectKey=taskmanager \
                            -Dsonar.sources=./backend \
                            -Dsonar.javascript.lcov.reportPaths=./backend/coverage/lcov.info \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONARQUBE_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Construire l’image Docker') {
            steps {
                sh """
                    docker build -t $DOCKER_IMAGE:$BUILD_NUMBER -t $DOCKER_IMAGE:latest backend/
                """
            }
        }

        stage('Pousser sur Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: "$DOCKER_CREDENTIALS", url: ""]) {
                    sh """
                        docker push $DOCKER_IMAGE:$BUILD_NUMBER
                        docker push $DOCKER_IMAGE:latest
                    """
                }
            }
        }

        stage('Déploiement sur TEST') {
            steps {
                withEnv(["KUBECONFIG=$KUBECONFIG_PATH"]) {
                    sh """
                        helm upgrade --install taskmanager-backend /var/lib/jenkins/taskmanager-backend-k3s \
                            --set image.repository=$DOCKER_IMAGE \
                            --set image.tag=$BUILD_NUMBER \
                            -n test --create-namespace
                    """
                }
            }
        }

        stage('Approval for Production') {
            steps {
                input message: "Déployer en production ?"
            }
        }

        stage('Déploiement sur PRODUCTION') {
            steps {
                withEnv(["KUBECONFIG=$KUBECONFIG_PATH"]) {
                    sh """
                        helm upgrade --install taskmanager-backend /var/lib/jenkins/taskmanager-backend-k3s \
                            --set image.repository=$DOCKER_IMAGE \
                            --set image.tag=$BUILD_NUMBER \
                            -n taskmanager --create-namespace
                    """
                }
            }
        }
    }

    post {
        always {
            junit 'backend/test-results/results.xml'
        }
    }
}
