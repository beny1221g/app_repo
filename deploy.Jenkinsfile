pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
                - name: jenkins-agent
                  image: beny14/dockerfile_agent:latest
                  command:
                    - cat
                  tty: true
              restartPolicy: Never
            '''
        }
    }

    parameters {
        string(name: 'PYTHON_IMAGE_NAME', defaultValue: 'beny14/python_app:latest', description: 'Name of the Python Docker image')
        string(name: 'PYTHON_BUILD_NUMBER', defaultValue: '', description: 'Build number of the Python Docker image to deploy')
        string(name: 'NGINX_IMAGE_NAME', defaultValue: 'beny14/nginx_static:latest', description: 'Name of the Nginx Docker image')
        string(name: 'NGINX_BUILD_NUMBER', defaultValue: '', description: 'Build number of the Nginx Docker image to deploy')
    }

    stages {
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Applying Kubernetes configurations"

                    // Ensure the namespace exists
                    sh 'kubectl create namespace jenkins || true'

                    // Apply Helm charts
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

        // Uncomment if needed for debugging
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