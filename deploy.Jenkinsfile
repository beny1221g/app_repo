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
        aws_region = "us-east-2"
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:deploy_bz"
        git_repo_url = "https://github.com/beny1221g/k8s.git"
        kubeconfig_path = "/root/.kube/config"
        namespace = "bz-appy"
    }

    stages {
        stage('Setup Tools') {
            steps {
                script {
                    container('install-tools') {
                        sh '''
                        apt-get update
                        apt-get install -y unzip curl git
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip awscliv2.zip
                        ./aws/install -i /usr/local/aws-cli -b /usr/local/bin
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl /usr/local/bin/
                        '''
                    }
                }
            }
        }

        stage('Download Deployment Files') {
            steps {
                container('install-tools') {
                    script {
                        echo "Cloning GitHub repository into /home/jenkins/agent/workspace/app_deploy/k8s"
                        sh '''
                            # Check if the directory already exists, and remove it if so
                            if [ -d "/home/jenkins/agent/workspace/app_deploy/k8s" ]; then
                                rm -rf /home/jenkins/agent/workspace/app_deploy/k8s
                            fi
                            # Clone the repository into the subdirectory
                            git clone ${git_repo_url} /home/jenkins/agent/workspace/app_deploy/k8s
                            echo "Listing the directory structure of /home/jenkins/agent/workspace/app_deploy/k8s"
                            ls -R /home/jenkins/agent/workspace/app_deploy/k8s
                            # Verify the contents of the directory tree to find the YAML files
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('install-tools') {
                    script {
                        echo "Applying Kubernetes YAML files for NGINX deployment"
                        sh '''
                        # Clean up old resources
                        kubectl delete -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-deployment.yaml --namespace ${namespace} || true
                        kubectl delete -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-hpa.yaml --namespace ${namespace} || true
                        kubectl delete -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-ingress.yaml --namespace ${namespace} || true
                        kubectl delete -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-service.yaml --namespace ${namespace} || true

                        # Apply new configurations
                        kubectl apply -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-deployment.yaml --namespace ${namespace}
                        kubectl apply -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-hpa.yaml --namespace ${namespace}
                        kubectl apply -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-ingress.yaml --namespace ${namespace}
                        kubectl apply -f /home/jenkins/agent/workspace/app_deploy/k8s/nginx-deployment/nginx-service.yaml --namespace ${namespace}
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
