pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        CLUSTER_NAME   = 'tim-mastery-v2-cluster'
        NAMESPACE      = 'ecom-app' // Change this if you have a specific app namespace
    }

    stages {
        stage('Pre-Flight Checks') {
            steps {
                echo 'Checking local tooling dependencies on Jenkins Agent...'
                sh 'aws --version'
                sh 'kubectl version --client'
                sh 'helm version'
            }
        }

        stage('Connect to EKS') {
            steps {
                echo "Generating kubeconfig for ${CLUSTER_NAME}..."
                // This assumes the identity of the attached EC2 IAM Role automatically
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
                
                echo "Testing cluster connectivity..."
                sh "kubectl get nodes"
            }
        }

        stage('App Validation / Test') {
            steps {
                echo 'Running application tests...'
                // Insert your unit tests or linting steps here
                // e.g., sh 'npm test' or sh 'python -m pytest'
            }
        }

        stage('Deploy to EKS') {
            steps {
	        dir('layer4-app-code') {
		    echo "Ensuring namespace ${NAMESPACE} exists..."
                    sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml |kubectl apply -f -"
                    echo "Deploying application to cluster..."
                // Example using Helm (Standard Track B practice)
                //  sh "helm upgrade --install my-app ./charts/my-app --namespace ${NAMESPACE} --v=5"
                
                // Example using raw manifests if you aren't using Helm yet:
                    sh "kubectl apply -f sample-app.yaml --namespace ${NAMESPACE}"
                    echo "Verifying rollout status..."
	            sh "kubectl rollout status deployment/echo-deployment --namespace ${NAMESPACE}"
            }
        }
    }
}

    post {
        success {
            echo 'Pipeline completed successfully! Workload is live.'
        }
        failure {
            echo 'Pipeline failed. Check the logs above for details.'
        }
    }
}
