pipeline {
    agent { label 'ec2-fleet-bz2' }

    options {
        timeout(time: 5, unit: 'MINUTES') // Sets a timeout for the entire pipeline
    }

    environment {
        aws_region = "us-east-2"                          // AWS region for EKS and SNS
        ecr_registry = "023196572641.dkr.ecr.us-east-2.amazonaws.com" // AWS ECR registry URL
        ecr_repo = "${ecr_registry}/beny14/aws_repo"      // ECR repository path
        image_tag_p = "python_app:${BUILD_NUMBER}"        // Tag for Python application image
        image_tag_n = "nginx_static:${BUILD_NUMBER}"      // Tag for NGINX static content image
        cluster_name = "eks-X10-prod-01"                  // EKS cluster name
        kubeconfig_path = "/root/.kube/config"            // Path to kubeconfig file in the container
        namespace = "bz-appy"                             // Kubernetes namespace for deployment
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:deploy_bz" // SNS topic for notifications
        git_repo_url = "https://github.com/beny1221g/k8s.git"          // Git repository URL for Helm charts
        localHelmPath = "${WORKSPACE}/k8s/nginx/nginx-app" // Path to Helm chart package
    }

    stages {
        stage('Configure kubectl') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                        credentialsId: 'aws'
                    ]
                ]) {
                    script {
                        echo "Configuring kubectl to use EKS cluster"
                        sh '''
                            aws eks --region ${aws_region} update-kubeconfig --name ${cluster_name}
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Ensuring cleanup of old resources in ${namespace} namespace"
                    sh '''
                        helm install nginx-static-release ${localHelmPath} --namespace ${namespace} --debug
                    '''
                }
            }
        }

        stage('Notify Deployment') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                        credentialsId: 'aws'
                    ]
                ]) {
                    script {
                        echo "Sending deployment notification"
                        sh '''
                            aws sns publish --topic-arn ${sns_topic_arn} --message "Deployment of ${image_tag_n} and ${image_tag_p} completed successfully."
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up resources..."
            // Cleanup code here, if necessary
        }
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Please check logs."
        }
    }
}
