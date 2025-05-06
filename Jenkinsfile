pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'yacine78/taskmanager'
        DOCKER_CREDENTIALS = 'credential-dockerhub'
        SONARQUBE_ENV = 'SonarQubeServer'
        SONARQUBE_TOKEN = credentials('sonarqube-token-id')
    }

    stages {
        stage('Cloner le code') {
            steps {
                git branch: 'main', url: 'https://github.com/Yacine781/taskmanager.git'
            }
        }

        stage('Tests & couverture') {
            steps {
                dir('backend') {
                    // Installation des dépendances
                    sh 'npm install'

                    // Exécution des tests + génération du rapport JUnit
                    sh 'npx nyc mocha --reporter mocha-junit-reporter --reporter-options mochaFile=./test-results/results.xml'

                    // Génération du rapport de couverture au format lcov
                    sh 'npx nyc report --reporter=text-lcov > coverage.lcov'
                }
            }
        }

        stage('Analyse SonarQube') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=taskmanager \
                        -Dsonar.sources=./backend \
                        -Dsonar.javascript.lcov.reportPaths=./backend/coverage.lcov \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONARQUBE_TOKEN
                    """
                }
            }
        }

        stage('Construire l’image Docker') {
            steps {
                script {
                    sh 'docker build -t $DOCKER_IMAGE:$BUILD_NUMBER -f backend/Dockerfile .'
                }
            }
        }

        stage('Pousser sur Docker Hub') {
            steps {
                script {
                    withDockerRegistry([credentialsId: "$DOCKER_CREDENTIALS", url: ""]) {
                        sh 'docker push $DOCKER_IMAGE:$BUILD_NUMBER'
                    }
                }
            }
        }
    }

    post {
        always {
            // Publication du rapport de test JUnit
            junit 'backend/test-results/results.xml'
        }
    }
}