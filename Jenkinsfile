pipeline {
    agent any

    environment {
        AWS_REGION   = 'us-east-1'
        CLUSTER_NAME = 'tim-mastery-v2-cluster'
        
        // Dynamically assign Namespace and Approval requirement based on Git Tag vs Branch Push
        TARGET_ENV   = "${env.TAG_NAME ? 'prod' : 'dev'}"
        NAMESPACE    = "${env.TAG_NAME ? 'ecom-app-prod' : 'ecom-app-dev'}"
    }

    stages {
        stage('Pre-Flight & Env Resolution') {
            steps {
                echo "=========================================="
                echo " Target Environment : ${TARGET_ENV.toUpperCase()}"
                echo " Target Namespace   : ${NAMESPACE}"
                echo " Triggered By Tag   : ${TAG_NAME ?: 'None (Branch Push)'}"
                echo "=========================================="
                
                sh 'aws --version'
                sh 'kubectl version --client'
            }
        }

        stage('Connect to EKS') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
            }
        }

        stage('Deploy Preview & Run Analysis') {
            steps {
                // Ensure target namespace exists (ecom-app-dev or ecom-app-prod)
                sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

                // Apply manifests into the target namespace
                sh "kubectl apply -f ecom-rollout.yaml -f ecom-analysis.yaml --namespace ${NAMESPACE}"

                // Wait for analysis to complete / rollout to reach Paused or Healthy
                sh """
                  echo "Waiting for rollout in ${NAMESPACE}..."
                  until kubectl argo rollouts get rollout ecom-app -n ${NAMESPACE} | grep -E "Paused|Healthy"; do
                    echo "Analysis running... sleeping 10s"
                    sleep 10
                  done
                """
            }
        }

        stage('Promote to Production / Dev') {
            steps {
                script {
                    if (TARGET_ENV == 'prod') {
                        echo "PROD deployment detected! Requiring manual approval..."
                        
                        // Interactive manual gate for PROD only
                        input message: "Approve deployment to PRODUCTION (${NAMESPACE})?", ok: "Promote to Prod"
                        
                        sh "kubectl argo rollouts promote ecom-app --namespace ${NAMESPACE}"
                        sh "kubectl argo rollouts status ecom-app --namespace ${NAMESPACE} --watch=true"
                    } else {
                        echo "DEV deployment detected! Auto-promoting..."
                        
                        // Auto-promote for DEV
                        sh "kubectl argo rollouts promote ecom-app --namespace ${NAMESPACE}"
                        sh "kubectl argo rollouts status ecom-app --namespace ${NAMESPACE} --watch=true"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Successfully deployed to ${TARGET_ENV.toUpperCase()} namespace (${NAMESPACE})!"
        }
        failure {
            echo "Deployment to ${TARGET_ENV.toUpperCase()} failed."
        }
    }
}
