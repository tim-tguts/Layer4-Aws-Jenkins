pipeline {
    agent any

    environment {
        AWS_REGION   = 'us-east-1'
        CLUSTER_NAME = 'tim-mastery-v2-cluster'
    }

    stages {
        stage('Pre-Flight & Env Resolution') {
            steps {
                script {
                    if (env.TAG_NAME && env.TAG_NAME != '') {
                        env.TARGET_ENV = 'prod'
                        env.NAMESPACE  = 'ecom-app-prod'
                    } else {
                        env.TARGET_ENV = 'dev'
                        env.NAMESPACE  = 'ecom-app-dev'
                    }
                }

                echo "=========================================="
                echo " Target Environment : ${env.TARGET_ENV.toUpperCase()}"
                echo " Target Namespace   : ${env.NAMESPACE}"
                echo " Triggered By Tag   : ${env.TAG_NAME ?: 'None (Branch Push)'}"
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
                sh "kubectl create namespace ${env.NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

                // Apply manifests into the target namespace
                sh "kubectl apply -f ecom-services.yaml -f ecom-rollout.yaml -f ecom-analysis.yaml --namespace ${env.NAMESPACE}"

                // Wait for analysis to complete / rollout to reach Paused or Healthy
                sh """
                  echo "Waiting for rollout in ${env.NAMESPACE}..."
                  until kubectl argo rollouts get rollout ecom-app -n ${env.NAMESPACE} | grep -E "Paused|Healthy"; do
                    echo "Analysis running... sleeping 10s"
                    sleep 10
                  done
                """
            }
        }

        stage('Promote to Production / Dev') {
            steps {
                script {
                    if (env.TARGET_ENV == 'prod') {
                        echo "PROD deployment detected! Requiring manual approval..."
                        
                        // Interactive manual gate for PROD only
                        input message: "Approve deployment to PRODUCTION (${env.NAMESPACE})?", ok: "Promote to Prod"
                    } else {
                        echo "DEV deployment detected! Auto-promoting..."
                    }

                    sh "kubectl argo rollouts promote ecom-app --namespace ${env.NAMESPACE}"
                    sh "kubectl argo rollouts status ecom-app --namespace ${env.NAMESPACE} --watch=true"
                }
            }
        }
    }

    post {
        success {
            echo "Successfully deployed to ${env.TARGET_ENV.toUpperCase()} namespace (${env.NAMESPACE})!"
        }
        failure {
            echo "Deployment to ${env.TARGET_ENV ? env.TARGET_ENV.toUpperCase() : 'UNKNOWN'} failed."
        }
    }
}
