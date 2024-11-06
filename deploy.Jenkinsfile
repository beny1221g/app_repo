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
                    - -url
                    - http://k8s-bzjenkin-releasej-c663409355-6f66daf7dc73980b.elb.us-east-2.amazonaws.com:8080
                    - -name
                    - jenkins-agent
                    - -secret
                    - ${env.JENKINS_AGENT_SECRET}
                    - -workDir
                    - /home/jenkins/agent
                  tty: true
                - name: install-tools
                  image: ubuntu:latest
                  command:
                    - sleep
                    - "3600"
                  tty: true
                  volumeMounts:
                    - name: kube-config
                      mountPath: /root/.kube
              volumes:
                - name: kube-config
                  configMap:
                    name: kubeconfig
              restartPolicy: Never
            '''
        }
    }

    environment {
        aws_region = "us-east-2"
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:deploy_bz"
        git_repo_url = "https://github.com/beny1221g/k8s.git"
        kubeconfig_path = "/root/.kube/config"
        namespace = "bz-appy"
    }

    stages {
        stage('Setup Tools') {
            steps {
                container('install-tools') {
                    script {
                        echo "Installing AWS CLI, kubectl, and dependencies..."
                        sh '''
                        apt-get update
                        apt-get install -y unzip curl git

                        # Install AWS CLI v2
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip awscliv2.zip
                        ./aws/install -i /usr/local/aws-cli -b /usr/local/bin

                        # Install kubectl
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl /usr/local/bin/
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('install-tools') {
                    script {
                        echo "Deploying NGINX image to Kubernetes"
                        sh '''
                        # Apply the Deployment
                        kubectl apply -f /home/jenkins/agent/workspace/app_deploy/k8s/k8s/nginx/nginx-deployment.yaml --namespace=${namespace}

                        # Apply the Service
                        kubectl apply -f /home/jenkins/agent/workspace/app_deploy/k8s/k8s/nginx/nginx-service.yaml --namespace=${namespace}

                        # Apply the Ingress (optional)
                        kubectl apply -f /home/jenkins/agent/workspace/app_deploy/k8s/k8s/nginx/nginx-ingress.yaml --namespace=${namespace}

                        # Wait for the deployment to complete
                        kubectl rollout status deployment/nginx-deployment --namespace=${namespace}
                        '''
                    }
                }
            }
        }

        stage('Notify Deployment') {
            steps {
                script {
                    sh '''
                    aws sns publish --topic-arn ${sns_topic_arn} --message "Deployment of NGINX application completed successfully."
                    '''
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
