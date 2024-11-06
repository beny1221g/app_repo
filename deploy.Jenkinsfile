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

        stage('Ensure Permissions for Resources') {
            steps {
                container('install-tools') {
                    script {
                        echo "Ensuring permissions to manage resources in ${namespace} namespace"
                        sh '''
                        kubectl auth can-i delete hpa --namespace ${namespace}
                        kubectl auth can-i delete deployment --namespace ${namespace}
                        kubectl auth can-i delete service --namespace ${namespace}
                        kubectl auth can-i delete pvc --namespace ${namespace}
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
                # Delete old resources using Helm if they exist
                helm delete nginx-static-release --namespace ${namespace} || echo "No nginx-static-release to delete"
                helm delete python-app-release --namespace ${namespace} || echo "No python-app-release to delete"

                # Optionally, remove leftover resources directly using kubectl
                kubectl get hpa -n ${namespace} && kubectl delete hpa -n ${namespace} --all || echo "No HPA resources to delete"
                kubectl get deployment -n ${namespace} && kubectl delete deployment -n ${namespace} --all || echo "No deployments to delete"
                kubectl get service -n ${namespace} && kubectl delete service -n ${namespace} --all || echo "No services to delete"
                kubectl get pvc -n ${namespace} && kubectl delete pvc -n ${namespace} --all || echo "No PVCs to delete"
                '''

                echo "Installing/Upgrading Helm releases"
                sh '''
                # Install or upgrade Helm releases (NGINX and Python app)
                helm upgrade --install nginx-static-release ${localHelmPath} --namespace ${namespace} --set image.tag=${image_tag_n} --set replicas=1 --set hpa.enabled=true --create-namespace
                helm upgrade --install python-app-release ${localHelmPath} --namespace ${namespace} --set image.tag=${image_tag_p} --create-namespace
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
