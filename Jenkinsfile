pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'yacine78/taskmanager'
        DOCKER_CREDENTIALS = 'credential-dockerhub'
        SONARQUBE_ENV = 'Sonarqube'
        SONARQUBE_TOKEN = credentials('credential-sonarqube')
        SONAR_HOST_URL = 'http://192.168.27.66:9000/' // Ajoute l'URL de ton serveur SonarQube
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
                    sh 'npm install'
                    // Lancer le test logique basique
                    sh 'npx mocha backend/test/basic.test.js --reporter mocha-junit-reporter --reporter-options mochaFile=./test-results/results.xml'
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
            script {
                // Ce bloc doit être dans "script" et dans "post"
                junit 'backend/test-results/results.xml'
            }
        }
    }
}