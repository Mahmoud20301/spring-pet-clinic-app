pipeline {
    agent any
    tools {
        maven 'maven-3.9.11'
    }
    environment {
        SONAR_HOST_URL = "http://sonarqube:9000"
    }
    stages {
        stage("Build") {
            steps {
                echo "Cleaning workspace and checking out code"
                deleteDir()
                checkout scm
                echo "Executing build"
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
        stage("Deploy to Nexus") {
            steps {
                echo "Uploading artifact to Nexus"
                withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIALS', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    script {
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: 'nexus:8081',
                            repository: 'spring-app',
                            credentialsId: 'NEXUS_CREDENTIALS',
                            groupId: 'org.springframework.samples',
                            version: '3.5.0-SNAPSHOT',
                            artifacts: [[
                                artifactId: 'spring-boot-starter-parent',
                                classifier: '',
                                file: 'target/spring-petclinic-*.jar',
                                type: 'jar'
                            ]]
                        )
                    }
                }
            }
        }
    }
    post {
        failure {
            echo "Pipeline failed. Check credentials, repository permissions, and Maven configuration."
        }
        success {
            echo "Pipeline completed successfully."
        }
    }
}
