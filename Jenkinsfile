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
                sh "mvn clean install -DskipTests"
            }
        }
        stage("Test") {
            steps {
                echo "Running SonarQube analysis"
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh "mvn test sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.projectKey=my-app -Dsonar.host.url=$SONAR_HOST_URL"
                }
            }
        }
        stage("Deploy to Nexus") {
            steps {
                echo "Uploading artifact to Nexus"
                withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIALS', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    script {
                        // Detect project version
                        def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                        echo "Project version detected: ${version}"

                        // Determine repository based on version
                        def nexusRepo = version.endsWith("SNAPSHOT") ? "maven-snapshots" : "maven-releases"
                        echo "Deploying to Nexus repository: ${nexusRepo}"

                        // Detect the built jar file dynamically
                        def jarFile = sh(script: "ls target/*.jar | head -n 1", returnStdout: true).trim()
                        echo "Deploying jar: ${jarFile}"

                        // Upload using Nexus Artifact Uploader
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: 'nexus:8081',  // container name in same Docker network
                            repository: nexusRepo,
                            credentialsId: 'NEXUS_CREDENTIALS',
                            groupId: 'org.springframework.samples',
                            version: version,
                            artifacts: [[
                                artifactId: 'spring-petclinic',
                                classifier: '',
                                file: jarFile,
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
