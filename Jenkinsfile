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

        stage('Deploy Preview & Run Analysis') {
            steps {
                echo "Listing files in the repo root:"
                sh "pwd"
                sh "ls -la"

                echo "Ensuring namespace ${NAMESPACE} exists..."
                sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

                echo "Deploying Argo Rollout and Analysis manifests to cluster..."
                sh "kubectl apply -f ecom-rollout.yaml -f ecom-analysis.yaml --namespace ${NAMESPACE}"

                echo "Waiting for Preview deployment & Pre-Promotion Analysis to pass..."
                // Loop until status is "Paused" (analysis passed, waiting for manual promotion)
                sh '''
                  echo "Waiting for pre-promotion analysis to complete and rollout to reach Paused state..."
                  until kubectl argo rollouts get rollout ecom-app -n ${NAMESPACE} | grep -E "Paused|Healthy"; do
                    echo "Analysis still running... sleeping 10s"
                    sleep 10
                  done
                  echo "Pre-promotion analysis completed successfully! Ready for promotion."
                '''
            }
        }

        stage('Promote to Production') {
            input {
                message "Promote new revision to Production (ecom-active)?"
                ok "Promote Cutover"
            }
            steps {
                echo "Manual approval received! Promoting traffic shift..."
                sh "kubectl argo rollouts promote ecom-app --namespace ${NAMESPACE}"
                
                echo "Verifying full cutover status..."
                sh "kubectl argo rollouts status ecom-app --namespace ${NAMESPACE} --watch=true"
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully! Workload is live in production.'
        }
        failure {
            echo 'Pipeline failed. Check the logs above for details.'
        }
    }
}
