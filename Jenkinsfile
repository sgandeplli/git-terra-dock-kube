pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = 'my-app'  // Define a name for the Docker image
        DOCKER_TAG = "${BUILD_NUMBER}"  // Use the Jenkins build number as the Docker tag
        MAVEN_HOME = '/opt/apache-maven-3.9.9' // Path to your Maven installation
        PATH = "${MAVEN_HOME}/bin:${env.PATH}" // Add Maven to the PATH
        DOCKER_HUB_USERNAME = 'sekhar1913'  // Set your Docker Hub username
        DOCKER_HUB_PASSWORD = credentials('docker-pass') // Docker Hub password from Jenkins credentials store
        GOOGLE_CREDENTIALS = credentials('gcp-sa')  // GCP Service Account Key for Terraform
        DEPLOY_YAML = 'deploy.yaml'  // Path to the Kubernetes deployment file
        // TF_STATE_BUCKET = 'your-terraform-state-bucket'  // Set your GCP bucket for Terraform state storage
        CLUSTER_NAME = 'cluster-11'  // GCP Cluster name (should be parameterized if needed)
        ZONE = 'us-west3-c'  // GCP Zone (should be parameterized if needed)
        PROJECT_ID = 'ferrous-upgrade-446608-k0'  // GCP project ID (should be parameterized if needed)
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                script {
                    if (!fileExists('first-demo-project')) {
                        sh 'git clone https://github.com/sekhar-dev/first-demo-project.git'
                    } else {
                        dir('first-demo-project') {
                            sh 'git reset --hard'
                            sh 'git pull origin master'
                        }
                    }
                }
            }
        }

        stage('Build Maven Project') {
            steps {
                script {
                    sh 'mvn clean install'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} .'
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    sh """
                    echo \$DOCKER_HUB_PASSWORD | docker login -u \$DOCKER_HUB_USERNAME --password-stdin
                    """
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    sh """
                    docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} ${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}:${DOCKER_TAG}
                    """
                    sh """
                    docker push ${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}:${DOCKER_TAG}
                    """
                }
            }
        }

        stage('Terraform: Initialize') {
            steps {
                script {
                    echo 'Initializing Terraform...'
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform: Apply Infrastructure') {
            steps {
                script {
                    echo 'Applying Terraform configurations to create GCP resources...'
                    // Ensure you're authenticated and have the necessary permissions to create resources
                    withCredentials([file(credentialsId: 'gcp-sa', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh 'gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS'
                        // Set the environment variable for Terraform GCP provider to use
                        sh 'export GOOGLE_APPLICATION_CREDENTIALS=$GOOGLE_APPLICATION_CREDENTIALS'
                        sh 'terraform apply -auto-approve'
                    }
                }
            }
        }

        stage('Update Deployment File') {
            steps {
                script {
                    echo 'Updating deployment YAML with the latest Docker image...'
                    // Update the image in the deployment.yaml file to match the built image
                    sh """
                    sed -i 's|image: .*|image: ${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}:${DOCKER_TAG}|' ${DEPLOY_YAML}
                    """
                }
            }
        }

        stage('Deploy Application') {
            steps {
                // Authenticate GCP and deploy the app using Kubernetes
                withCredentials([file(credentialsId: 'gcp-sa', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    script {
                        echo 'Authenticating with GCP...'
                        sh 'gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS'

                        // Get credentials for the Kubernetes cluster
                        sh """
                        gcloud container clusters get-credentials ${CLUSTER_NAME} --zone ${ZONE} --project ${PROJECT_ID}
                        """

                        // Apply the Kubernetes manifest to deploy the app
                        sh 'kubectl apply -f ${DEPLOY_YAML}'
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Build, Docker image creation, Docker push, Terraform infrastructure creation, and Kubernetes deployment successful!"
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
