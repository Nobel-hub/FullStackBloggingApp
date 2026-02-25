pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_REGION     = 'us-east-1'
        EKS_CLUSTER    = 'blogging-app-cluster'
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

        // ────────────────────────────────────────────────
        //     Modern EKS authentication using AWS CLI
        // ────────────────────────────────────────────────
        stage('K8s Deploy and Run') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-eks-creds',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh '''
                        # Set AWS region
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        
                        # Generate fresh kubeconfig for this build
                        aws eks update-kubeconfig --name ${EKS_CLUSTER}
                        
                        # Apply manifests and wait for rollout
                        kubectl apply -f deployment-service.yml -n webapps
                        kubectl rollout status deployment/bloggingapp-deployment \
                            -n webapps \
                            --timeout=180s
                    '''
                }
            }
        }

        stage('K8s Verify') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-eks-creds',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh '''
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        aws eks update-kubeconfig --name ${EKS_CLUSTER}
                        
                        echo "Pods in webapps namespace:"
                        kubectl get pods -n webapps -o wide
                        
                        echo -e "\\nServices in webapps namespace:"
                        kubectl get svc -n webapps
                    '''
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
        failure {
            mail to: 'nobeldhakal01@gmail.com',
                 subject: "Jenkins Build FAILED: ${JOB_NAME} #${BUILD_NUMBER}",
                 body: """Hi,

The pipeline '${JOB_NAME}' (#${BUILD_NUMBER}) failed.

Check the console output here: ${BUILD_URL}console

Regards,
Jenkins"""
        }
    }
}