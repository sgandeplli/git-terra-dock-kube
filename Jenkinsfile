pipeline {
    agent any

     parameters {
        string(name: 'CLUSTER_NAME', defaultValue: 'my-cluster11', description: 'The GKE cluster name')
        string(name: 'ZONE', defaultValue: 'us-west3', description: 'The GCP zone')
    }

    environment {
        REPO_URL = 'https://github.com/sgandeplli/git-terra-dock-kube.git' // Replace with your GitHub repo URL
        IMAGE_NAME = 'sekhar1913/final' // Replace with your Docker Hub image name
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials' // Jenkins credentials ID for Docker Hub
        
        DEPLOY_YAML = 'deploy.yaml' // Path to your Kubernetes deployment manifest
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Cloning the repository...'
                git branch: 'main', url: "${REPO_URL}"
            }
        }

        stage('Build and Push Docker Image') {
    steps {
        script {
            def imageTag = "latest-${env.BUILD_NUMBER}" // Use BUILD_NUMBER to tag the image uniquely
            def fullImageName = "${IMAGE_NAME}:${imageTag}"

            echo "Building the Docker image: ${fullImageName}"
            sh "docker build -t ${fullImageName} ."

            echo 'Pushing the Docker image to Docker Hub...'
            withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", 
                                             usernameVariable: 'DOCKER_USERNAME', 
                                             passwordVariable: 'DOCKER_PASSWORD')]) {
                sh """
                echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                docker push ${fullImageName}
                """
            }

            // Export the full image name for use in the deployment manifest
            env.FULL_IMAGE_NAME = fullImageName
        }
    }
}

        stage('Run Terraform') {
            steps {
                echo 'Running Terraform to create the cluster...'
                    withCredentials([file(credentialsId: 'gcp-sa', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                         sh 'terraform init'
                        sh 'terraform apply -auto-approve'
                }        
            }
        }

        stage('Update Deployment File') {
            steps {
                script {
                    echo 'Updating deployment YAML with the latest Docker image...'
                    sh """
                    sed -i 's|image: .*|image: ${env.FULL_IMAGE_NAME}|' ${DEPLOY_YAML}
                    """
                }
            }
        }

        stage('Deploy Application') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh 'gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS'
                    sh "gcloud container clusters get-credentials ${params.CLUSTER_NAME} --zone ${params.ZONE} --project ${params.PROJECT_ID}"

                    // Create or update Kubernetes deployment
                    sh 'kubectl apply -f deploy.yaml'
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for errors.'
        }
    }
}
