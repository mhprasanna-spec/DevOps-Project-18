pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        IMAGE             = "prasanna369/ultimate-cicd"   // fixed: matches deployment manifest
        AWS_DEFAULT_REGION = "us-east-1"
    }
    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/mhprasanna-spec/DevOps-Project-18.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -f spring-boot-app/pom.xml clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        mvn -f spring-boot-app/pom.xml \
                        -Dmaven.multiModuleProjectDirectory=spring-boot-app \
                        sonar:sonar
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    dir('spring-boot-app') {
                        sh """
                            docker build -t ${IMAGE}:${BUILD_NUMBER} .
                            docker tag  ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )
                ]) {
                    sh """
                        echo \$PASS | docker login -u \$USER --password-stdin
                        docker push ${IMAGE}:${BUILD_NUMBER}
                        docker push ${IMAGE}:latest
                        # Clean up local images to save disk space
                        docker rmi ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest || true
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']
                ]) {
                    script {
                        env.KUBECONFIG = KUBECONFIG_FILE

                        sh """
                            # Use a temp copy so IMAGE_TAG placeholder is preserved
                            # in the original file for future builds
                            cp spring-boot-app-manifests/deployment.yml /tmp/deployment-${BUILD_NUMBER}.yml

                            # Replace placeholder in the temp copy only
                            sed -i "s|IMAGE_TAG|${BUILD_NUMBER}|g" /tmp/deployment-${BUILD_NUMBER}.yml

                            # Validate substitution worked
                            echo "--- Image line after substitution ---"
                            grep "image:" /tmp/deployment-${BUILD_NUMBER}.yml

                            # Apply manifests (kubectl apply is idempotent, no delete needed)
                            kubectl apply -f /tmp/deployment-${BUILD_NUMBER}.yml
                            kubectl apply -f spring-boot-app-manifests/service.yml

                            # Wait for rollout to complete — fails pipeline if pods don't come up
                            kubectl rollout status deployment/spring-boot-app --timeout=120s

                            # Clean up temp file
                            rm -f /tmp/deployment-${BUILD_NUMBER}.yml
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build #${BUILD_NUMBER} deployed successfully — ${IMAGE}:${BUILD_NUMBER}"
        }
        failure {
            echo "❌ Build #${BUILD_NUMBER} failed — check logs above"
        }
        always {
            // Clean workspace to avoid stale files affecting next run
            cleanWs()
        }
    }
}
