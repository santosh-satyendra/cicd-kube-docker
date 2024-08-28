pipeline {
    agent any

    environment {
        registry = "satya263/vproappdock"
        registryCredential = "dockerhub"
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Archiving WAR files...'
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
                    echo 'Checkstyle analysis completed.'
                }
            }
        }

        stage('Code Analysis with SonarQube') {
            environment {
                scannerHome = tool 'mysonarscanner4'
            }
            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/classes/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = "${registry}:V${BUILD_NUMBER}"
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                sh "docker rmi ${dockerImage}"
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${dockerImage} --namespace prod"
            }
        }
    }
}
