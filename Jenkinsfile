pipeline {

    agent any

    environment {

        // GCP
        GCP_PROJECT = 'healthcare-488108'
        GCP_REGION  = 'asia-south1-c'

        // Artifact Registry
        GAR_REPOSITORY = 'devops-repo'
        IMAGE_NAME     = 'java-app'

        // GKE
        GKE_CLUSTER = 'devops-gke'
        GKE_ZONE    = 'asia-south1-c'

        // Jenkins Credential
        GOOGLE_APPLICATION_CREDENTIALS =
            credentials('service-account')
    }

    stages {

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                    -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('GCP Authentication') {
            steps {
                sh '''
                    gcloud auth activate-service-account \
                      --key-file="$GOOGLE_APPLICATION_CREDENTIALS"

                    gcloud config set project "$GCP_PROJECT"
                '''
            }
        }

        stage('Docker Authentication') {
            steps {
                sh '''
                    gcloud auth configure-docker \
                      ${GCP_REGION}-docker.pkg.dev \
                      --quiet
                '''
            }
        }

        stage('Push Image to Artifact Registry') {
            steps {
                sh '''
                    docker tag \
                    ${IMAGE_NAME}:${BUILD_NUMBER} \
                    ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}

                    docker push \
                    ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('GKE Authentication') {
            steps {
                sh '''
                    gcloud container clusters get-credentials \
                      "$GKE_CLUSTER" \
                      --zone "$GKE_ZONE" \
                      --project "$GCP_PROJECT"
                '''
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh '''
                    sed "s|IMAGE_PLACEHOLDER|${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}|g" \
                    k8s/deployment.yaml > k8s/deployment-final.yaml

                    kubectl apply -f k8s/deployment-final.yaml

                    kubectl rollout status deployment/java-app
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl get pods
                    kubectl get services
                    kubectl get deployment
                '''
            }
        }
    }

    post {

        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD pipeline failed!'
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
        }
    }
}
