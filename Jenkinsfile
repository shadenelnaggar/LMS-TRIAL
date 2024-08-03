pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "shadenelnaggar/devops-project-trial"
        DOCKER_CREDENTIALS = '579edcb8-3533-4a89-afea-ee7c9312ab80'
        AWS_CREDENTIALS = 'c382d1de-a305-423a-b22f-daece7e0a0be'
        TERRAFORM_DIR = "C:\\Users\\Hp\\Downloads\\terraform_1.9.3_windows_amd64\\terraform.exe"
        AWS_DIR = "C:\\Users\\Hp\\AppData\\Roaming\\Python\\Python312\\Scripts\\aws.cmd"
        KUBECTL_DIR = "C:\\Users\\Hp\\kubectl.exe"
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
                     bat "kubectl config view --kubeconfig ${KUBECONFIG_PATH}"
                 }
             }
         }
         
         stage('Update Kubeconfig') {
           steps {
                script {
                     withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS}"]]) {
                        bat """
                            ${AWS_DIR} eks --region %AWS_DEFAULT_REGION% update-kubeconfig --name TeamTwoCluster-${env.BUILD_NUMBER} --kubeconfig ${KUBECONFIG_PATH}
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
                         ${KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\pv.yaml
                         ${KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\pvc.yaml
                         ${KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\deployment.yaml
                         ${KUBECTL_DIR} --kubeconfig ${KUBECONFIG_PATH} apply -f ${env.WORKSPACE}\\service.yaml
                         """
                    }
                 }
             }
         }

         stage('Deploy Ingress') {
            steps {
                bat '${KUBECTL_DIR} apply -f ${env.WORKSPACE}\\ingress.yaml'
            }
        }
    }

    post {
        always {
            // Clean up workspace after build
            cleanWs()
            script {
                echo "Pipeline failed. Destroying the infrastructure..."
                dir("${env.TERRAFORM_CONFIG_PATH}") {
                    bat """${env.TERRAFORM_DIR} destroy -auto-approve"""
                }
            }
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            script {
                echo "Pipeline failed. Destroying the infrastructure..."
                dir("${env.TERRAFORM_CONFIG_PATH}") {
                    bat """${env.TERRAFORM_DIR} destroy -auto-approve"""
                }
            }
        }
    }
}
