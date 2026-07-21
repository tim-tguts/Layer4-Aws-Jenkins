pipeline {
    agent any

    environment {
        AWS_REGION   = 'us-east-1'
        CLUSTER_NAME = 'tim-mastery-v2-cluster'
        NAMESPACE    = 'ecom-app'
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
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
                
                echo "Testing cluster connectivity..."
                sh "kubectl get nodes"
            }
        }

        stage('App Validation / Test') {
            steps {
                echo 'Running application tests...'
            }
        }

        stage('Deploy to EKS') {
            steps {
                echo "Listing files in the repo root:"
                sh "pwd"
                sh "ls -la"

                echo "Ensuring namespace ${NAMESPACE} exists..."
                sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

                echo "Deploying Argo Rollout and Analysis manifests to cluster..."
                // Apply all manifests in layer4/ecom-app/
                sh "kubectl apply -f layer4/ecom-app/ --namespace ${NAMESPACE}"

                echo "Verifying Argo Rollout status..."
                // Monitor the rollout until it finishes promotion/analysis
                sh "kubectl argo rollouts status rollout/ecom-app --namespace ${NAMESPACE}"
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
