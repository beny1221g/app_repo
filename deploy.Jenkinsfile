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
        localHelmPath = "${WORKSPACE}/nginx-chart/k8s/nginx/nginx-app"  // Path to Helm chart package
    }

    stages {
        stage('Setup Tools') {
            steps {
                script {
                    echo "Installing required tools in the 'install-tools' container"
                    container('install-tools') {
                        sh '''
                        set -e  # Stop on any error
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

        stage('Configure kubectl') {
            steps {
                container('install-tools') {
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
                container('install-tools') {
                    script {
                        echo "Ensuring cleanup of old resources in ${namespace} namespace"
                        sh '''

                        helm upgrade --install nginx-static-release ${localHelmPath} --namespace ${namespace}  --set hpa.enabled=true
                        # -- helm upgrade --install python-app-release ${localHelmPath} --namespace ${namespace} --set image.tag=${image_tag_p}
                        '''
                    }
                }
            }
        }

        stage('Notify Deployment') {
            steps {
                script {
                    echo "Sending deployment notification"
                    sh '''
                    aws sns publish --topic-arn ${sns_topic_arn} --message "Deployment of ${image_tag_n} and ${image_tag_p} completed successfully."
                    '''
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
