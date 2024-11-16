pipeline {
    agent { label 'ec2-fleet-bz2' }

    options {
        timeout(time: 5, unit: 'MINUTES') // Sets a timeout for the entire pipeline
    }

    environment {
        aws_region = "us-east-2"
        ecr_registry = "023196572641.dkr.ecr.us-east-2.amazonaws.com"
        ecr_repo = "${ecr_registry}/beny14/aws_repo"
        image_tag_p = "python_app:${BUILD_NUMBER}"
        image_tag_n = "nginx_static:${BUILD_NUMBER}"
        cluster_name = "eks-X10-prod-01"
        kubeconfig_path = "${WORKSPACE}/.kube/config" // Update to use workspace directory
        namespace = "bz-appy"
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:deploy_bz"
        git_repo_url = "https://github.com/beny1221g/k8s.git"
        localHelmPath = "${WORKSPACE}/k8s/nginx/nginx-app"
    }

    stages {
        stage('AWS Configure') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                    credentialsId: 'aws'
                ]]) {
                    script {
                        // Configure AWS without interpolating secrets in Groovy string
                        sh """
                            aws configure set aws_access_key_id ${env.AWS_ACCESS_KEY_ID}
                            aws configure set aws_secret_access_key ${env.AWS_SECRET_ACCESS_KEY}
                            aws configure set region ${aws_region}
                        """

                        // Update kubeconfig to point to EKS cluster
                        sh """
                            aws eks --region ${aws_region} update-kubeconfig --name ${cluster_name} --kubeconfig ${kubeconfig_path}
                        """
                    }
                }
            }
        }

        stage('Check Kubernetes Connectivity') {
            steps {
                script {
                    // Check if Kubernetes is reachable using kubectl
                    echo "Checking Kubernetes cluster connectivity"
                    sh """
                        kubectl cluster-info --kubeconfig ${kubeconfig_path}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying resources using Helm"
                    sh """
                        helm upgrade --install nginx-bz ${localHelmPath} --namespace ${namespace} --kubeconfig ${kubeconfig_path}
                    """
                }
            }
        }

        stage('Notify Deployment') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                    credentialsId: 'aws'
                ]]) {
                    script {
                        echo "Sending deployment notification"
                        sh """
                            aws sns publish --topic-arn ${sns_topic_arn} --message "Deployment of ${image_tag_n} and ${image_tag_p} completed successfully."
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up resources..."
        }
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Please check logs."
        }
    }
}
