pipeline {
    agent any
    tools {
        maven 'maven-3.9.11'  
    }
    stages {
        stage("Build") {
            steps {
                echo "Executing build"
                sh "mvn clean install"
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
