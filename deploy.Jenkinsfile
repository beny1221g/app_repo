pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: jenkins-agent
              namespace: jenkins
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
                    - http://10.100.105.105:30080/computer/jenkins-agent/slave-agent.jnlp
                    - -secret
                    - ea1ec6413419076fbe9c88e36bce4519d02d29854741d6a0544c7b370bc428f5
                    - -workDir
                    - /home/jenkins/agent
                  tty: true
              restartPolicy: Never
            '''
        }
    }

    options {
        timeout(time: 2, unit: 'MINUTES') // Agent connection timeout
    }

    parameters {
        string(name: 'PYTHON_IMAGE_NAME', defaultValue: 'beny14/python_app:latest', description: 'Python Docker image name')
        string(name: 'PYTHON_BUILD_NUMBER', defaultValue: '', description: 'Python Docker image build number')
        string(name: 'NGINX_IMAGE_NAME', defaultValue: 'beny14/nginx_static:latest', description: 'Nginx Docker image name')
        string(name: 'NGINX_BUILD_NUMBER', defaultValue: '', description: 'Nginx Docker image build number')
    }

    stages {
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Applying Kubernetes configurations"

                    // Ensure the namespace exists
                    sh 'kubectl create namespace jenkins || true'

                    // Apply Helm charts with custom image tags
                    sh """
                    helm upgrade --install app-release ./k8s/app/app-chart \
                      --namespace jenkins \
                      --set image.tag=${params.PYTHON_BUILD_NUMBER}

                    helm upgrade --install nginx-release ./k8s/nginx/nginx-chart \
                      --namespace jenkins \
                      --set image.tag=${params.NGINX_BUILD_NUMBER}
                    """

                    echo "Kubernetes configurations applied"
                }
            }
        }

        // Optional debugging stages
        /*
        stage('Check Pod Status') {
            steps {
                script {
                    echo "Checking pod status"
                    sh 'kubectl get pods -n jenkins'
                    sh 'kubectl describe pods -n jenkins'
                }
            }
        }

        stage('Port Forwarding') {
            steps {
                script {
                    echo "Attempting to port-forward"
                    sh 'kubectl port-forward svc/nginx-service 4000:4000 -n jenkins'
                }
            }
        }
        */
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
