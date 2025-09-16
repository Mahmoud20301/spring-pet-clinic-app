pipeline {
    agent any
    tools {
        maven 'maven-3.9.11'  
    }
    environment {
        // Cache Maven dependencies inside workspace
        MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2/repository"
    }
    stages {
        stage("Build") {
            steps {
                echo "Executing build"
                sh "mvn dependency:go-offline -B"
                sh "mvn clean install -B"
            }
        }
        stage("Test") {
            steps {
                echo "Executing test"
            }
        }
        stage("Deploy") {
            steps {
                echo "Executing deploy"
            }
        }
    }
}
