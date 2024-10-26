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
                    - ${env.JENKINS_AGENT_SECRET} // Ensure this is set properly
                    - -workDir
                    - /home/jenkins/agent
                  tty: true
              restartPolicy: Never
            '''
        }
    }

    options {
        timeout(time: 5, unit: 'MINUTES') // Increase if necessary
    }

    parameters {
        string(name: 'PYTHON_IMAGE_NAME', defaultValue: 'beny14/python_app:latest', description: 'Python Docker image name')
        string(name: 'PYTHON_BUILD_NUMBER', defaultValue: '', description: 'Python Docker image build number')
        string(name: 'NGINX_IMAGE_NAME', defaultValue: 'beny14/nginx_static:latest', description: 'Nginx Docker image name')
        string(name: 'NGINX_BUILD_NUMBER', defaultValue: '', description: 'Nginx Docker image build number')
        string(name: 'JENKINS_AGENT_SECRET', defaultValue: '', description: 'Jenkins Agent Secret') // Add a parameter for the secret
    }

    stages {
        stage('Setup') {
            steps {
                script {
                    echo "Setting up namespace and ensuring kubectl is installed"

                    // Check if kubectl is installed; if not, download and install it
                    sh '''
                    if ! command -v kubectl &> /dev/null; then
                        echo "kubectl not found, installing..."
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl" && \
                        chmod +x ./kubectl && \
                        mv ./kubectl /home/jenkins/kubectl && \
                        export PATH=$PATH:/home/jenkins
                    else
                        echo "kubectl is already installed"
                    fi

                    kubectl create namespace bz-jenkins || true
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Applying Kubernetes configurations"
                    if (!params.PYTHON_BUILD_NUMBER?.trim() || !params.NGINX_BUILD_NUMBER?.trim()) {
                        error("Build numbers for Python and Nginx cannot be empty")
                    }
                    try {
                        sh '''
                        helm upgrade --install app-release ./k8s/app/app-chart \
                          --namespace bz-jenkins \
                          --set image.tag=${params.PYTHON_BUILD_NUMBER} || exit 1

                        helm upgrade --install nginx-release ./k8s/nginx/nginx-chart \
                          --namespace bz-jenkins \
                          --set image.tag=${params.NGINX_BUILD_NUMBER} || exit 1
                        '''
                    } catch (Exception e) {
                        error "Failed to deploy applications: ${e.message}"
                    }
                    echo "Kubernetes configurations applied"
                }
            }
        }

        stage('Port Forwarding') {
            steps {
                script {
                    echo "Attempting to port-forward"
                    sh 'kubectl port-forward svc/nginx-service 4000:4000 -n bz-jenkins'
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



// pipeline {
//     agent {
//         kubernetes {
//             yaml '''
//             apiVersion: v1
//             kind: Pod
//             metadata:
//               name: jenkins-agent
//               namespace: jenkins
//             spec:
//               containers:
//                 - name: jenkins-agent
//                   image: beny14/dockerfile_agent:latest
//                   command:
//                     - java
//                     - -jar
//                     - /usr/share/jenkins/agent.jar
//                   args:
//                     - -jnlpUrl
//                     - http://192.168.49.2:30080/computer/jenkins-agent/slave-agent.jnlp
//                     - -secret
//                     - ${env.JENKINS_AGENT_SECRET} // Use an environment variable for the secret
//                     - -workDir
//                     - /home/jenkins/agent
//                   tty: true
//               restartPolicy: Never
//               containers:
//                 - name: jenkins-agent
//                   image: beny14/dockerfile_agent:latest
//                   command:
//                     - java
//                     - -jar
//                     - /usr/share/jenkins/agent.jar
//                   args:
//                     - -jnlpUrl
//                     - http://192.168.49.2:30080/computer/jenkins-agent/slave-agent.jnlp
//                     - -secret
//                     - ${env.JENKINS_AGENT_SECRET} // Use an environment variable for the secret
//                     - -workDir
//                     - /home/jenkins/agent
//                   tty: true
//               restartPolicy: Never
//             '''
//         }
//     }
//
//     options {
//         timeout(time: 2, unit: 'MINUTES') // Agent connection timeout
//     }
//
//     parameters {
//         string(name: 'PYTHON_IMAGE_NAME', defaultValue: 'beny14/python_app:latest', description: 'Python Docker image name')
//         string(name: 'PYTHON_BUILD_NUMBER', defaultValue: '', description: 'Python Docker image build number')
//         string(name: 'NGINX_IMAGE_NAME', defaultValue: 'beny14/nginx_static:latest', description: 'Nginx Docker image name')
//         string(name: 'NGINX_BUILD_NUMBER', defaultValue: '', description: 'Nginx Docker image build number')
//         string(name: 'JENKINS_AGENT_SECRET', defaultValue: '', description: 'Jenkins Agent Secret') // Add a parameter for the secret
//     }
//
//     stages {
//         stage('Setup') {
//             steps {
//                 script {
//                     echo "Setting up namespace"
//
//                     // Ensure the namespace exists
//                     sh 'kubectl create namespace jenkins || true'
//                 }
//             }
//         }
//
//         stage('Deploy to Kubernetes') {
//             steps {
//                 script {
//                     echo "Applying Kubernetes configurations"
//
//                     // Validate build numbers
//                     if (!params.PYTHON_BUILD_NUMBER?.trim() || !params.NGINX_BUILD_NUMBER?.trim()) {
//                         error("Build numbers for Python and Nginx cannot be empty")
//                     }
//
//                     // Apply Helm charts with custom image tags
//                     try {
//                         sh """
//                         helm upgrade --install app-release ./k8s/app/app-chart \
//                           --namespace jenkins \
//                           --set image.tag=${params.PYTHON_BUILD_NUMBER} || exit 1
//
//                         helm upgrade --install nginx-release ./k8s/nginx/nginx-chart \
//                           --namespace jenkins \
//                           --set image.tag=${params.NGINX_BUILD_NUMBER} || exit 1
//                         """
//                     } catch (Exception e) {
//                         error "Failed to deploy applications: ${e.message}"
//                     }
//
//                     echo "Kubernetes configurations applied"
//                 }
//             }
//         }
//
//         // Optional debugging stages
//         /*
//         stage('Check Pod Status') {
//             steps {
//                 script {
//                     echo "Checking pod status"
//                     sh 'kubectl get pods -n jenkins'
//                     sh 'kubectl describe pods -n jenkins'
//                 }
//             }
//         }
//
//         stage('Port Forwarding') {
//             steps {
//                 script {
//                     echo "Attempting to port-forward"
//                     sh 'kubectl port-forward svc/nginx-service 4000:4000 -n jenkins'
//                 }
//             }
//         }
//         */
//     }
//
//     post {
//         always {
//             echo "Pipeline completed"
//         }
//         failure {
//             echo "Pipeline failed"
//         }
//     }
// }
