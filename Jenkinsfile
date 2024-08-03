pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "shadenelnaggar/devops-project"
        DOCKER_CREDENTIALS = '579edcb8-3533-4a89-afea-ee7c9312ab80'
        AWS_CREDENTIALS = 'c382d1de-a305-423a-b22f-daece7e0a0be'
        TERRAFORM_DIR = tool name: 'terraform', type: 'terraform'
        AWS_DIR = tool name: 'aws-cli', type: 'aws'
        KUBECTL_DIR = tool name: 'kubectl', type: 'kubectl'
        TERRAFORM_CONFIG_PATH = "${env.WORKSPACE}\\terraform"
        KUBECONFIG_PATH = "${env.WORKSPACE}\\kubeconfig"
    }

    stages {
        stage('Clone Git Repository') {
            steps {
                echo "Cloning the Git repository"
                // Add your git clone command here if needed
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image with build number as tag
                    docker.build("${DOCKER_IMAGE}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo "Pushing Docker image ${DOCKER_IMAGE}:${env.BUILD_NUMBER} to Docker Hub"
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        bat """
                        echo Logging into Docker Hub...
                        echo %DOCKER_PASSWORD% | docker login -u %DOCKER_USERNAME% --password-stdin
                        docker tag ${DOCKER_IMAGE}:${env.BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Terraform Init') {
            steps {
                script {
                    // Initialize Terraform
                    dir("${env.TERRAFORM_CONFIG_PATH}") {
                        bat """${env.TERRAFORM_DIR} init"""
                    }
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                script {
                    // Generate and show the Terraform execution plan
                    dir("${env.TERRAFORM_CONFIG_PATH}") {
                        bat """${env.TERRAFORM_DIR} plan"""
                    }
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                script {
                    // Apply the Terraform plan to deploy the infrastructure
                    dir("${env.TERRAFORM_CONFIG_PATH}") {
                        bat """${env.TERRAFORM_DIR} apply -auto-approve"""
                    }
                }
            }
        }

        stage('Verify Kubeconfig Path') {
            steps {
                script {
                    echo "KUBECONFIG path is set to: ${env.KUBECONFIG_PATH}"
                    bat "${env.KUBECTL_DIR} config view --kubeconfig ${KUBECONFIG_PATH}"
                }
            }
        }

        stage('Update Kubeconfig') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"]]) {
                        bat """
                            ${env.AWS_DIR} eks --region %AWS_DEFAULT_REGION% update-kubeconfig --name TeamTwoCluster-${env.BUILD_NUMBER} --kubeconfig ${KUBECONFIG_PATH}
                        """
                    }
                }
            }
        }

        stage('Deploy Kubernetes Resources') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"]]) {
                        bat """
                        ${env.KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\pv.yaml
                        ${env.KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\pvc.yaml
                        ${env.KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\deployment.yaml
                        ${env.KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\service.yaml
                        """
                    }
                }
            }
        }

        stage('Deploy Ingress') {
            steps {
                sh 'kubectl apply -f ingress.yaml'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            script {
                echo "Pipeline failed. Destroying the infrastructure..."
                dir("${env.TERRAFORM_CONFIG_PATH}") {
                    sh "${env.TERRAFORM_DIR} destroy -auto-approve"
                }
            }
        }
        success {
            echo 'Pipeline completed successfully.'
        }
    }
}
