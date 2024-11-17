pipeline {
    agent { label 'ec2-fleet-bz2' }

    options {
        timeout(time: 2, unit: 'MINUTES')
    }

    environment {
        aws_region = "us-east-2"
        ecr_registry = "023196572641.dkr.ecr.us-east-2.amazonaws.com"
        ecr_repo = "${ecr_registry}/beny14/aws_repo"
        image_tag_n = "nginx_static:latest"
        image_tag_p = "python_app:latest"
        cluster_name = "eks-X10-prod-01"
        kubeconfig_path = "${WORKSPACE}/.kube/config"
        namespace = "bz-appy"
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:deploy_bz"
        localHelmPath_n = "${WORKSPACE}/nginx/nginx-app"
        localHelmPath_p = "${WORKSPACE}/app/app-chart"
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
                        sh """
                            aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                            aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                            aws configure set region ${aws_region}

                            # Update kubeconfig for EKS cluster
                            aws eks --region ${aws_region} update-kubeconfig --name ${cluster_name} --kubeconfig ${kubeconfig_path}
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying resources using Helm"
                    sh """
                        helm upgrade --install nginx-bz ${localHelmPath_n} \
                            --namespace ${namespace} \
                            --kubeconfig ${kubeconfig_path} \
                            --set image.repository=${ecr_registry}/nginx_static,image.tag=latest

                        helm upgrade --install app-bz ${localHelmPath_p} \
                            --namespace ${namespace} \
                            --kubeconfig ${kubeconfig_path} \
                            --set image.repository=${ecr_registry}/python_app,image.tag=latest
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
                            aws sns publish --topic-arn ${sns_topic_arn} \
                                --message "Deployment of ${image_tag_n} and ${image_tag_p} to Kubernetes namespace ${namespace} completed successfully." \
                                --subject "Kubernetes Deployment Notification"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up temporary files..."
            cleanWs() // Clean up workspace
        }
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Please check logs for details."
        }
    }
}
