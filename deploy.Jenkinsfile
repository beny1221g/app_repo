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

    stages {
        stage('Setup Helm and kubectl') {
            steps {
                script {
                    echo "Setting up kubectl and Helm if not installed"

                    // Ensure kubectl is installed
                    sh '''
                    if ! command -v kubectl &> /dev/null; then
                        echo "Installing kubectl"
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl" && \
                        chmod +x kubectl && mv kubectl /usr/local/bin/
                    fi
                    '''

                    // Ensure Helm is installed
                    sh '''
                    if ! command -v helm &> /dev/null; then
                        echo "Installing Helm"
                        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
                    fi
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying Nginx with DockerHub image"

                    // Deploy the Nginx chart, assuming values.yaml is set up correctly
                    sh '''
                    helm upgrade --install nginx-release ./k8s/nginx/nginx-chart \
                      --namespace bz-appy \
                      --set nginx.image=beny14/nginx_static \
                      --set nginx.replicas=1
                    '''
                }
            }
        }

        stage('Port Forwarding') {
            steps {
                script {
                    echo "Attempting to port-forward"
                    sh 'kubectl port-forward svc/nginx-service 4000:4000 -n bz-appy'
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
