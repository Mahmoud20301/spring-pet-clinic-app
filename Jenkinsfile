pipeline {
    agent any
    tools {
        maven 'maven-3.9.11'
    }
    environment {
        SONAR_HOST_URL = "http://localhost:9002" 
    }
    stages {
        stage("Build") {
            steps {
                echo "Cleaning workspace before build"
                deleteDir() // cleans the entire workspace
                checkout scm // restores project files
                echo "Executing build"
                sh "mvn clean install"
            }
        }
        stage("Test") {
            steps {
                echo "Executing test with SonarQube"
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh "mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.projectKey=my-app -Dsonar.host.url=$SONAR_HOST_URL"
                }
            }
        }
        stage("Deploy") {
            steps {
                echo "Executing deploy"
            }
        }
    }
}
