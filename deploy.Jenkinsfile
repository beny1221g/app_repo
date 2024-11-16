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
        kubeconfig_path = "/root/.kube/config"
        namespace = "bz-appy"
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:deploy_bz"
        git_repo_url = "https://github.com/beny1221g/k8s.git"
        localHelmPath = "${WORKSPACE}/k8s/nginx/nginx-app"
    }

    stages {
        stage('Configure kubectl') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                    credentialsId: 'aws'
                ]]) {
                    script {
                        echo "Configuring kubectl to use EKS cluster"
                        sh """
                            aws eks --region ${aws_region} update-kubeconfig --name ${cluster_name} --kubeconfig ${kubeconfig_path}
                        """
                    }
                }
            }
        }

        stage('Prepare Namespace') {
            steps {
                script {
                    echo "Ensuring namespace ${namespace} exists"
                    sh """
                        kubectl get namespace ${namespace} || kubectl create namespace ${namespace}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying resources using Helm"
                    sh """
                        helm install nginx-static ${localHelmPath} --namespace ${namespace} --debug --kubeconfig ${kubeconfig_path}
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
