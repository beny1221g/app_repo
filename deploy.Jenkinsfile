pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: jenkins-agent
              namespace: bz-jenkins
            spec:
              initContainers:
                - name: init-permissions
                  image: busybox
                  command: ['sh', '-c', 'chmod -R 777 /home/jenkins/agent']
                  volumeMounts:
                    - name: jenkins-home
                      mountPath: /home/jenkins/agent
              containers:
                - name: jenkins-agent
                  image: beny14/dockerfile_agent:latest
                  command:
                    - java
                    - -jar
                    - /usr/share/jenkins/agent.jar
                  args:
                    - -url
                    - http://k8s-bzjenkin-releasej-c663409355-6f66daf7dc73980b.elb.us-east-2.amazonaws.com:8080
                    - -name
                    - jenkins-agent
                    - -secret
                    - ${env.JENKINS_AGENT_SECRET}
                    - -workDir
                    - /home/jenkins/agent
                  tty: true
                  securityContext:
                    runAsUser: 1000
                    fsGroup: 1000
                - name: install-tools
                  image: ubuntu:20.04
                  command: ['sleep', 'infinity']
                  tty: true
              volumes:
                - name: jenkins-home
                  emptyDir: {}
              restartPolicy: Never
            '''
        }
    }

    options {
        timeout(time: 5, unit: 'MINUTES')  // Sets a timeout for the entire pipeline
    }

    environment {
        aws_region = "us-east-2"  // AWS region for EKS and SNS
        ecr_registry = "023196572641.dkr.ecr.us-east-2.amazonaws.com"  // AWS ECR registry URL
        ecr_repo = "${ecr_registry}/beny14/aws_repo"  // ECR repository path
        image_tag_p = "python_app:${BUILD_NUMBER}"  // Tag for Python application image
        image_tag_n = "nginx_static:${BUILD_NUMBER}"  // Tag for NGINX static content image
        cluster_name = "eks-X10-prod-01"  // EKS cluster name
        kubeconfig_path = "/root/.kube/config"  // Path to kubeconfig file in the container
        namespace = "bz-appy"  // Kubernetes namespace for deployment
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:deploy_bz"  // SNS topic for notifications
        git_repo_url = "https://github.com/beny1221g/k8s.git"  // Git repository URL for Helm charts
        localHelmPath = "${WORKSPACE}/nginx-chart/k8s/nginx/nginx-chart"  // Path to Helm chart package
    }

    stages {
        stage('Setup Tools') {
            steps {
                script {
                    echo "Installing required tools in the 'install-tools' container"
                    container('install-tools') {
                        sh '''
                        # Update and install essential packages
                        apt-get update
                        apt-get install -y unzip curl git

                        # Download and install AWS CLI
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip awscliv2.zip
                        ./aws/install -i /usr/local/aws-cli -b /usr/local/bin

                        # Download and install kubectl
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl /usr/local/bin/

                        # Download and install Helm
                        curl -LO https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
                        tar -zxvf helm-v3.9.0-linux-amd64.tar.gz
                        mv linux-amd64/helm /usr/local/bin/
                        chmod +x /usr/local/bin/helm
                        '''
                    }
                }
            }
        }

        stage('Generate Helm Chart Directory') {
            steps {
                container('install-tools') {
                    script {
                        echo "Setting up directory structure for Helm chart"
                        sh '''
                            # Create the Helm chart directory
                            mkdir -p ${WORKSPACE}/nginx-chart
                            ls -l ${WORKSPACE}/nginx-chart  # List contents to verify
                        '''
                    }
                }
            }
        }

        stage('Download Helm Chart') {
            steps {
                container('install-tools') {
                    script {
                        echo "Cloning Helm chart repository"
                        sh '''
                            # Clone the repository to the Jenkins workspace
                            git clone ${git_repo_url} /home/jenkins/agent/workspace/app_deploy/nginx-chart
                            # Check the structure of the cloned directory
                            ls -R /home/jenkins/agent/workspace/app_deploy/nginx-chart
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('install-tools') {
                    script {
                        echo "Deploying Helm chart to Kubernetes namespace: ${namespace}"
                        sh '''
                                echo "Release nginx-bz exists; upgrading..."
                                helm install nginx-bz ${localHelmPath} -n ${namespace}
                        '''
                    }
                }
            }
        }


    }

    post {
        success {
            echo "Deployment to EKS completed successfully."
            sendSNSNotification("SUCCESS", "nginx-Deployment to EKS completed successfully for ${env.deployment_name}")
        }
        failure {
            echo "Deployment failed. Check logs for details."
            sendSNSNotification("FAILURE", "Deployment failed for ${env.deployment_name}. Check logs for details.")
        }
    }
}

// Helper function to send SNS notifications
def sendSNSNotification(status, message) {
    script {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
            credentialsId: 'aws']]) {
            container('install-tools') {
                sh """
                    aws sns publish \
                        --region "${env.aws_region}" \
                        --topic-arn "${env.sns_topic_arn}" \
                        --message "Deployment Status: ${status}\\nMessage: ${message}" \
                        --subject "Deployment ${status}: ${env.deployment_name}"
                """
            }
        }
    }
}
