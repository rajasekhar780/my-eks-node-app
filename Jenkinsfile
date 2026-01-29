pipeline {
    agent any
    
    environment {
        // 1. These IDs must match the "ID" field you set in Jenkins Credentials UI
        DOCKER_HUB_CREDS = credentials('docker-hub-creds') 
        K8S_SECRET_ID    = 'k8s-config-text' 
        
        // 2. Change these to your actual Docker Hub details
        DOCKER_USER      = "your-dockerhub-username"
        IMAGE_NAME       = "nodejs-eks-app"
        
        // This helps track versions (e.g., nodejs-eks-app:1, nodejs-eks-app:2)
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs() // Starts with a fresh folder
            }
        }

        stage('Checkout from GitHub') {
            steps {
                // This pulls the code from the repo you connected to this job
                checkout scm 
            }
        }

        stage('Build & Push to Docker Hub') {
            steps {
                sh """
                    # Build the image using the Dockerfile in the repo
                    docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_USER}/${IMAGE_NAME}:latest
                    
                    # Login and Push
                    echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_CREDS_USR} --password-stdin
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to EKS (US-East-1)') {
            steps {
                // This uses your copy-pasted Secret Text
                withCredentials([string(credentialsId: "${K8S_SECRET_ID}", variable: 'KUBE_TEXT')]) {
                    sh '''
                        # 1. Create temporary config file
                        echo "$KUBE_TEXT" > kubeconfig
                        
                        # 2. Update the YAML file to use the new image version
                        # This looks for 'image:' in nodejsapp.yaml and updates it
                        sed -i "s|image:.*|image: ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}|g" nodejsapp.yaml
                        
                        # 3. Apply the changes to the cluster in the US
                        kubectl --kubeconfig=kubeconfig apply -f nodejsapp.yaml
                        
                        # 4. Verify deployment status
                        kubectl --kubeconfig=kubeconfig rollout status deployment/nodeapp-deployment
                        
                        # 5. Securely remove the config file
                        rm kubeconfig
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh "docker logout" // Clear credentials from the Jenkins server
        }
    }
}
