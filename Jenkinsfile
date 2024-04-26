pipeline {
    agent any

    tools {
        // Define JDK and Maven versions
        jdk 'jdk17'
        maven 'maven3'
    }
  
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_DEFAULT_REGION = "ap-south-1"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/shankarraojadav/register-app_k8s.git'
            }
        }

        stage('Compile') {
            steps {
                // Compile the Maven project
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                // Run tests using Maven
                sh "mvn test"
            }
        }

        stage('Build') {
            steps {
                // Package the Maven project
                sh "mvn package"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh "$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=register \
                        -Dsonar.projectKey=register \
                        -Dsonar.java.binaries=."
                }
            }
        }

        stage('Quality Gate') {
            steps {
                // Wait for the SonarQube analysis to complete and check the Quality Gate
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
               script {
                   // Build and tag Docker image
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t shankarjad/register:latest ."
                    }
               }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                // Scan Docker image for vulnerabilities
                sh "trivy image --format table -o trivy-image-report.html shankarjad/register:latest "
            }
        }
        
        stage('Push Docker Image') {
            steps {
               script {
                   // Push Docker image to Docker registry
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker push shankarjad/register:latest"
                    }
               }
            }
        }
        
       stage('Deploying to EKS') {
            steps{
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials-id', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
    script {
                        sh """
                            aws eks update-kubeconfig --name practice-cluster --region ap-south-1
                            kubectl apply -f deployment.yaml -n webapps
                        """
                    }
}
            }
        }
    }
}
