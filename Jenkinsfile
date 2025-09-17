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
                echo "Deploying artifact to Nexus"
                withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIALS', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    script {
                        // Get project version
                        def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                        
                        // Select correct repository based on version
                        def nexusRepo = version.endsWith("SNAPSHOT") ? "maven-snapshots" : "maven-releases"

                        echo "Deploying version ${version} to repository ${nexusRepo}"

                        // Deploy using container hostname
                        sh """
                            mvn deploy \
                            -Dnexus.username=$NEXUS_USER \
                            -Dnexus.password=$NEXUS_PASS \
                            -DaltDeploymentRepository=nexus::default::http://nexus:8081/repository/${nexusRepo}/
                        """
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
