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
              containers:
                - name: jenkins-agent
                  image: beny14/dockerfile_agent:latest
                  command:
                    - java
                    - -jar
                    - /usr/share/jenkins/agent.jar
                  args:
                    - -jnlpUrl
                    - http://k8s-bzjenkin-releasej-c663409355-6f66daf7dc73980b.elb.us-east-2.amazonaws.com:8080/computer/jenkins-agent/slave-agent.jnlp
                    - -secret
                    - ${env.JENKINS_AGENT_SECRET}
                    - -workDir
                    - /home/jenkins/agent
                  tty: true
                  securityContext:
                    runAsUser: 0
                    allowPrivilegeEscalation: true
              restartPolicy: Never
            '''
        }
    }

    options {
        timeout(time: 5, unit: 'MINUTES')
    }

    parameters {
        string(name: 'JENKINS_AGENT_SECRET', defaultValue: '', description: 'Jenkins Agent Secret')
    }

    environment {
        aws_region = "us-east-2"
        ecr_registry = "023196572641.dkr.ecr.us-east-2.amazonaws.com"
        ecr_repo = "${ecr_registry}/beny14/aws_repo"

        image_tag_p = "python_app:${BUILD_NUMBER}"
        image_tag_n = "nginx_static:${BUILD_NUMBER}"

        cluster_name = "eks-X10-prod-01"
        kubeconfig_path = "~/.kube/config"
        namespace = "bz-appy"
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:osher-nginx-deployment"
        helm_chart_path = "/home/ec2-user/nginx-chart-0.1.0.tgz"
        git_repo_url = "https://github.com/beny1221g/k8s.git"
    }

    stages {

        stage('Setup AWS CLI, Helm, and kubectl') {
            steps {
                script {
                    echo "Setting up AWS CLI, kubectl, and Helm if not installed"

                    // Ensure unzip is installed
                    sh '''
                    if ! command -v unzip &> /dev/null; then
                        echo "Installing unzip"
                        apt-get update && apt-get install -y unzip
                    fi
                    '''

                    // Ensure AWS CLI is installed
                    sh '''
                    if ! command -v aws &> /dev/null; then
                        echo "Installing AWS CLI"
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip awscliv2.zip
                        ./aws/install -i /usr/local/aws-cli -b /usr/local/bin
                    fi
                    '''

                    // Ensure kubectl is installed
                    sh '''
                    if ! command -v kubectl &> /dev/null; then
                        echo "Installing kubectl"
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl" && \
                        chmod +x kubectl && mv kubectl /home/jenkins/kubectl
                        export PATH=$PATH:/home/jenkins
                    fi
                    '''

                    // Ensure Helm is installed
                    sh '''
                    if ! command -v helm &> /dev/null; then
                        echo "Installing Helm"
                        curl -LO https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz && \
                        tar -zxvf helm-v3.9.0-linux-amd64.tar.gz && \
                        mv linux-amd64/helm /home/jenkins/helm && \
                        chmod +x /home/jenkins/helm
                        export PATH=$PATH:/home/jenkins
                    fi
                    '''
                }
            }
        }

        stage('AWS Configure') {
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
                        // Configure AWS CLI with the provided credentials and region
                        sh """
                            aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                            aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                            aws configure set region ${aws_region}
                        """
                    }
                }
            }
        }

        stage('Fetch Helm Chart') {
            steps {
                script {
                    if (!fileExists(env.helm_chart_path)) {
                        echo "Helm chart not found. Cloning from Git..."
                        sh """
                            git clone ${git_repo_url} /tmp/nginx
                            cp /tmp/nginx/nginx-chart/nginx-app-0.1.0.tgz ${env.helm_chart_path}
                        """
                    } else {
                        echo "Helm chart found locally."
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        export HELM_DRIVER=configmap
                        helm install nginx-chart ${env.helm_chart_path} -n ${namespace} --kubeconfig ${kubeconfig_path}
                    """
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

def sendSNSNotification(status, message) {
    sh """
        aws sns publish \
            --region ${env.aws_region} \
            --topic-arn ${env.sns_topic_arn} \
            --message "Deployment Status: ${status}\\nMessage: ${message}" \
            --subject "Deployment ${status}: ${env.deployment_name}"
    """
}
