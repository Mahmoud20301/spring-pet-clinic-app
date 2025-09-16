pipeline {
    agent any
    tools {
        jdk 'java-17'          
        maven 'maven-3.9.11'
    }
    environment {
        SONAR_HOST_URL = "http://sonarqube:9000"
    }
    stages {
        stage("Build") {
            steps {
                echo "Cleaning workspace before build"
                deleteDir()
                checkout scm
                echo "Executing Maven build"
                sh "mvn clean install"
            }
        }
        stage("Test") {
            steps {
                echo "Running SonarQube analysis"
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh "mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.projectKey=my-app -Dsonar.host.url=$SONAR_HOST_URL"
                }
            }
        }
        stage("Deploy") {
            steps {
                echo "Building Docker image using Dockerfile"
                sh "docker build -t mahmoudmo123/spring-pipeline:${GIT_COMMIT} ."

                echo "Logging in to Docker Hub"
                withCredentials([usernamePassword(credentialsId: 'docker-token',
                                                 usernameVariable: 'DOCKER_USER',
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                }

                echo "Pushing Docker image to Docker Hub"
                sh "docker push mahmoudmo123/spring-pipeline:${GIT_COMMIT}"
            }
        }
    }
    post {
        always {
            echo "Cleaning up workspace"
            deleteDir()
        }
    }
}
