pipeline {
    agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: jenkins-agent
            image: jenkins-agent:latest
            command:
            - cat
            tty: true
        '''
    }
  }


    parameters {
        string(name: 'PYTHON_IMAGE_NAME', defaultValue: 'beny14/python_app', description: 'Name of the Python Docker image')
        string(name: 'PYTHON_BUILD_NUMBER', defaultValue: '', description: 'Build number of the Python Docker image to deploy')
        string(name: 'NGINX_IMAGE_NAME', defaultValue: 'beny14/nginx_static', description: 'Name of the Nginx Docker image')
        string(name: 'NGINX_BUILD_NUMBER', defaultValue: '', description: 'Build number of the Nginx Docker image to deploy')
    }

    stages {
        stage('Download Artifacts') {
            steps {
                copyArtifacts(projectName: 'app_build', filter: 'k8s/*.yaml', target: 'k8s')
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Applying Kubernetes configurations"

                    // Apply deployments
                    sh "kubectl set image deployment/app-deployment app=${PYTHON_IMAGE_NAME}:latest -n benyz"
                    sh "kubectl set image deployment/nginx-deployment nginx=${NGINX_IMAGE_NAME}:latest -n benyz"

                    sh 'kubectl apply -f k8s/nginx-service.yaml -n benyz'
                    sh 'kubectl apply -f k8s/nginx-ingress.yaml -n benyz'

                    echo "Kubernetes configurations applied"
                }
            }
        }

        stage('Check Pod Status') {
            steps {
                script {
                    echo "Checking pod status"
                    sh 'kubectl get pods -n benyz'
                    sh 'kubectl describe pods -n benyz'
                }
            }
        }

        stage('Port Forwarding') {
            steps {
                script {
                    echo "Attempting to port-forward"
                    sh 'kubectl port-forward svc/nginx-service 4000:4000 -n benyz'
                    // kubectl port-forward pod/app-deployment-86b9fd7945-bvwc6 5000:5000 -n benyz
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
