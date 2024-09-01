pipeline {
    agent {
        kubernetes {
            inheritFrom 'app_deploy_32-t9vg' // Ensure this name matches the pod template defined in Jenkins
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

                    // Apply deployments
                    sh "helm install app-release ./k8s/app/app-chart -n jenkins" // Adjust namespace if needed
                    sh "helm install nginx-release ./k8s/nginx/nginx-chart -n jenkins" // Adjust namespace if needed

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
                    sh 'kubectl get pods -n jenkins' // Adjust namespace if needed
                    sh 'kubectl describe pods -n jenkins' // Adjust namespace if needed
                }
            }
        }

        stage('Port Forwarding') {
            steps {
                script {
                    echo "Attempting to port-forward"
                    sh 'kubectl port-forward svc/nginx-service 4000:4000 -n jenkins' // Adjust namespace if needed
                    // kubectl port-forward pod/app-deployment-86b9fd7945-bvwc6 5000:5000 -n jenkins // Adjust namespace if needed
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
