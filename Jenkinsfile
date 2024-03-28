pipeline {
    agent any

    environment {
        registry = "jugal109/vprofile-app-jenkins"
        registryCredential = 'dockerhub'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Archiving artifacts...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('Code Analysis with Checkstyle') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Checkstyle analysis result'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }
        }
        
        stage('Deploy Docker Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                sh "docker rmi $registry:$BUILD_NUMBER"
                sh "docker rmi $registry:latest"
            }
        }

        stage('Code Analysis with SonarQube') {
            environment {
                scannerHome = tool 'mysonarscanner4'
            }
            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh """${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"""
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
