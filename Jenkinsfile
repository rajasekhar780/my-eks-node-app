pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDS = credentials('docker-hub-creds') 
        K8S_SECRET_ID    = 'k8s-config-text' 
        
        DOCKER_USER      = "rajasekhar142"
        IMAGE_NAME       = "my-docker-app"
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
                    # Create a folder named 'build-assets' (or whatever name you prefer)
                    mkdir -p my-custom-folder
                    
                    # Optional: Move files into it or just log it
                    echo "Folder created for build ${env.BUILD_NUMBER}" > my-custom-folder/build-info.txt
                    
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
