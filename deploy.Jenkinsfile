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
              volumeMounts:
                - name: kubeconfig-volume
                  mountPath: /root/.kube
            - name: install-tools
              image: ubuntu:20.04
              command: ['sleep', 'infinity']
              tty: true
          volumes:
            - name: jenkins-home
              emptyDir: {}
            - name: kubeconfig-volume
              secret:
                secretName: your-kubeconfig-secret # Replace with your secret name
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
        kubeconfig_path = "/root/.kube/config" // Make sure this path is correct
        namespace = "bz-appy"
        sns_topic_arn = "arn:aws:sns:us-east-2:023196572641:osher-nginx-deployment"
        git_repo_url = "https://github.com/beny1221g/k8s.git"
        localHelmPath = "${WORKSPACE}/nginx-chart/nginx-chart-0.1.0.tgz" // Updated path to use Jenkins workspace
    }

    stages {
        stage('Setup Tools') {
            steps {
                script {
                    echo "Installing required packages"
                    container('install-tools') {
                        sh '''
                        apt-get update
                        apt-get install -y unzip curl git

                        # Install AWS CLI
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip awscliv2.zip
                        ./aws/install -i /usr/local/aws-cli -b /usr/local/bin

                        # Install kubectl
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl /usr/local/bin/

                        # Install Helm
                        curl -LO https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
                        tar -zxvf helm-v3.9.0-linux-amd64.tar.gz
                        mv linux-amd64/helm /usr/local/bin/
                        chmod +x /usr/local/bin/helm
                        '''
                    }
                }
            }
        }

        stage('Generate Helm Chart') {
            steps {
                container('install-tools') {
                    script {
                        echo "Creating necessary directories..."
                        sh '''
                            mkdir -p ${WORKSPACE}/nginx-chart
                            echo "Current directory contents:"
                            ls -l ${WORKSPACE}/nginx-chart
                        '''
                    }
                }
            }
        }


        stage('Download Helm Chart') {
    steps {
        container('install-tools') {
            script {
                echo "Cloning repository for Helm chart..."
                sh '''
                    git clone ${git_repo_url} /home/jenkins/agent/workspace/app_deploy/nginx-chart
                    echo "Contents of the directory after cloning:"
                    ls -l /home/jenkins/agent/workspace/app_deploy/nginx-chart

                    # Check the actual structure
                    echo "Checking directory structure:"
                    ls -R /home/jenkins/agent/workspace/app_deploy/nginx-chart

                    # List contents before the deployment
                    echo "Contents of the deployment directory:"
                    ls -l /home/jenkins/agent/workspace/app_deploy/nginx-chart/k8s/nginx/nginx-chart
                '''
            }
        }
    }
}
        stage('Deploy to Kubernetes') {
            steps {
                container('install-tools') {
                    script {
                        echo "Attempting to install Helm chart from: ${localHelmPath}"

                        // Update the Helm install command
                        sh '''
                            export KUBECONFIG=${kubeconfig_path}  # Adjust the path as necessary
                            export HELM_DRIVER=configmap

                            # Check if the release already exists
                            if helm ls -n ${namespace} | grep -q nginx-bz; then
                                echo "Release nginx-bz already exists. Attempting to upgrade..."
                                helm upgrade nginx-bz ${localHelmPath} -n ${namespace} --install
                            else
                                echo "Installing new release nginx-bz..."
                                helm install nginx-bz ${localHelmPath} -n ${namespace}
                            fi
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
