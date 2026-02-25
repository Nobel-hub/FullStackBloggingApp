pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: 'main']],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/Nobel-hub/FullStackBloggingApp']]
                )
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=Blogging-app \
                        -Dsonar.projectKey=Blogging-app \
                        -Dsonar.java.binaries=target
                    """
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Publish Artifacts') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'maven-settings',
                    jdk: 'jdk17',
                    maven: 'maven3',
                    traceability: true
                ) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Docker Build and Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-creds') {
                        sh 'docker build -t nobeldhakal123/bloggingapp:v1 .'
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-creds') {
                        sh 'docker push nobeldhakal123/bloggingapp:v1'
                    }
                }
            }
        }

        stage('K8s Deploy and Run') {
            steps {
                // Using kubeconfig secret file
                withKubeConfig([credentialsId: 'kubeconfig-file']) {
                    sh "kubectl apply -f deployment-service.yml -n webapps"
                    sh "kubectl rollout status deployment/bloggingapp-deployment -n webapps"
                }
            }
        }

        stage('K8s Verify') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-file']) {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get service -n webapps"
                }
            }
        }
    }

    post {
        success {
            mail to: 'nobeldhakal01@gmail.com',
                 subject: "Jenkins Build SUCCESS: ${JOB_NAME} #${BUILD_NUMBER}",
                 body: """Hi,

Your pipeline '${JOB_NAME}' (#${BUILD_NUMBER}) has successfully deployed the application.

Check the details at: ${BUILD_URL}

K8s Deployment and verification completed successfully in namespace 'webapps'.

Regards,
Jenkins"""
        }
    }
}