pipeline {
    agent any
    
    environment {
        // These IDs must match the "ID" field you set in Jenkins Credentials UI
        DOCKER_HUB_CREDS = credentials('docker-hub-creds') 
        
        DOCKER_USER      = "rajasekhar142"
        IMAGE_NAME       = "my-docker-app"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs() 
            }
        }

        stage('Checkout from GitHub') {
            steps {
                checkout scm 
            }
        }

        stage('Build & Push to Docker Hub') {
            steps {
                sh """
                    # Create your custom folder
                    mkdir -p my-custom-folder
                    echo "Folder created for build ${env.BUILD_NUMBER}" > my-custom-folder/build-info.txt
                    
                    # Build and Tag
                    docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_USER}/${IMAGE_NAME}:latest
                    
                    # Login using Jenkins auto-generated variables (_USR and _PSW)
                    echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_CREDS_USR} --password-stdin
                    
                    # Push both tags
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to EKS (US-East-1)') {
            steps {
                // Using the Secret File credential ID you created
                withCredentials([file(credentialsId: 'k8s-config-file', variable: 'KUBECONFIG_PATH')]) {
                    sh """
                        # Update the image in your YAML
                        sed -i "s|image:.*|image: ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}|g" nodejsapp.yaml
                        
                        # Apply to Kubernetes using the path provided by Jenkins
                        kubectl --kubeconfig=${KUBECONFIG_PATH} apply -f nodejsapp.yaml
                        
                        # Wait for the rollout to finish
                        kubectl --kubeconfig=${KUBECONFIG_PATH} rollout status deployment/nodejs-app
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker logout" 
        }
    }
}
