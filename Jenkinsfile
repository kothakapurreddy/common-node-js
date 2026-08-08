pipeline {

    agent any

    environment {

        // ==============================
        // GCP
        // ==============================
        GCP_PROJECT = 'healthcare-488108'
        GCP_REGION  = 'asia-south1'

        // ==============================
        // Artifact Registry
        // ==============================
        GAR_REPOSITORY = 'devops-repo'
        IMAGE_NAME     = 'java-app'

        // ==============================
        // GKE
        // ==============================
        GKE_CLUSTER = 'devops-gke'
        GKE_ZONE    = 'asia-south1'

        // ==============================
        // Jenkins GCP Credential
        // ==============================
        GOOGLE_APPLICATION_CREDENTIALS =
            credentials('service-account')
    }

    stages {

        // ==========================================
        // 1. Checkout Code
        // ==========================================
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ==========================================
        // 2. Docker Build
        // ==========================================
        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    echo "Docker image:"
                    docker images | grep ${IMAGE_NAME}
                '''
            }
        }

        // ==========================================
        // 3. GCP Authentication
        // ==========================================
        stage('GCP Authentication') {
            steps {
                sh '''
                    echo "Authenticating with GCP..."

                    gcloud auth activate-service-account \
                      --key-file="$GOOGLE_APPLICATION_CREDENTIALS"

                    gcloud config set project "$GCP_PROJECT"

                    echo "Authenticated account:"
                    gcloud auth list

                    echo "Current project:"
                    gcloud config get-value project
                '''
            }
        }

        // ==========================================
        // 4. Docker → Artifact Registry Authentication
        // ==========================================
        stage('Docker Authentication') {
            steps {
                sh '''
                    echo "Configuring Docker authentication..."

                    gcloud auth configure-docker \
                      ${GCP_REGION}-docker.pkg.dev \
                      --quiet
                '''
            }
        }

        // ==========================================
        // 5. Push Docker Image
        // ==========================================
        stage('Push Image to Artifact Registry') {
            steps {
                sh '''
                    IMAGE_URI="${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}"

                    echo "Image URI:"
                    echo "$IMAGE_URI"

                    docker tag \
                      ${IMAGE_NAME}:${BUILD_NUMBER} \
                      "$IMAGE_URI"

                    echo "Pushing image..."

                    docker push "$IMAGE_URI"
                '''
            }
        }

        // ==========================================
        // 6. Verify Artifact Registry
        // ==========================================
        stage('Verify Artifact Registry') {
            steps {
                sh '''
                    echo "Checking Artifact Registry..."

                    gcloud artifacts docker images list \
                      ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY} \
                      --include-tags
                '''
            }
        }

        // ==========================================
        // 7. GKE Authentication
        // ==========================================
        stage('GKE Authentication') {
            steps {
                sh '''
                    echo "Getting GKE credentials..."

                    gcloud container clusters get-credentials \
                      "$GKE_CLUSTER" \
                      --zone "$GKE_ZONE" \
                      --project "$GCP_PROJECT"

                    echo "Checking GKE access..."

                    kubectl get nodes
                '''
            }
        }

        // ==========================================
        // 8. Deploy to GKE
        // ==========================================
        stage('Deploy to GKE') {
            steps {
                sh '''
                    IMAGE_URI="${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}"

                    echo "Deploying image:"
                    echo "$IMAGE_URI"

                    sed "s|IMAGE_PLACEHOLDER|${IMAGE_URI}|g" \
                      k8s/deployment.yaml \
                      > k8s/deployment-final.yaml

                    kubectl apply -f k8s/deployment-final.yaml

                    if [ -f k8s/service.yaml ]; then
                        kubectl apply -f k8s/service.yaml
                    fi

                    echo "Waiting for rollout..."

                    kubectl rollout status deployment/java-app
                '''
            }
        }

        // ==========================================
        // 9. Verify Deployment
        // ==========================================
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "===== PODS ====="
                    kubectl get pods -o wide

                    echo "===== DEPLOYMENT ====="
                    kubectl get deployment

                    echo "===== SERVICES ====="
                    kubectl get services
                '''
            }
        }
    }

    post {

        success {
            echo '========================================'
            echo 'CI/CD PIPELINE COMPLETED SUCCESSFULLY'
            echo 'Application deployed to GKE'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'CI/CD PIPELINE FAILED'
            echo 'Check the failed stage and logs'
            echo '========================================'
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
        }
    }
}
