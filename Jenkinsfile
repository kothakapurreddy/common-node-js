pipeline {

    agent any

    environment {

        // ==============================
        // GCP
        // ==============================
        GCP_PROJECT = 'healthcare-488108'
        GCP_REGION  = 'us-central1'

        // ==============================
        // Artifact Registry
        // ==============================
        GAR_REPOSITORY = 'devops-repo'
        IMAGE_NAME     = 'java-app'

        // ==============================
        // GKE - Regional Cluster
        // ==============================
        GKE_CLUSTER = 'devops-gke'
        GKE_REGION  = 'asia-south1'

        // ==============================
        // Jenkins GCP Credential
        // ==============================
        GOOGLE_APPLICATION_CREDENTIALS =
            credentials('gcp-service-account')
    }

    stages {


        // ==========================================
        // 2. Docker Build
        // ==========================================
        stage('Docker Build') {
            steps {
                sh '''
                    set -e

                    echo "Building Docker image..."

                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    echo "Docker image created:"
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
                    set -e

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
        // 4. Docker Authentication
        // ==========================================
        stage('Docker Authentication') {
            steps {
                sh '''
                    set -e

                    echo "Configuring Docker for Artifact Registry..."

                    gcloud auth configure-docker \
                      ${GCP_REGION}-docker.pkg.dev \
                      --quiet
                '''
            }
        }

        // ==========================================
        // 5. Push Image to Artifact Registry
        // ==========================================
        stage('Push Image to Artifact Registry') {
            steps {
                sh '''
                    set -e

                    IMAGE_URI="${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}"

                    echo "Image URI:"
                    echo "$IMAGE_URI"

                    docker tag \
                      ${IMAGE_NAME}:${BUILD_NUMBER} \
                      "$IMAGE_URI"

                    echo "Pushing image..."

                    docker push "$IMAGE_URI"

                    echo "Image pushed successfully."
                '''
            }
        }

        // ==========================================
        // 6. Verify Artifact Registry
        // ==========================================
        stage('Verify Artifact Registry') {
            steps {
                sh '''
                    set -e

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
                    set -e

                    echo "Checking GKE cluster status..."

                    gcloud container clusters describe \
                      "$GKE_CLUSTER" \
                      --region "$GKE_REGION" \
                      --project "$GCP_PROJECT" \
                      --format="value(status)"

                    echo "Getting GKE credentials..."

                    gcloud container clusters get-credentials \
                      "$GKE_CLUSTER" \
                      --region "$GKE_REGION" \
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
                    set -e

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

                    echo "Waiting for deployment rollout..."

                    kubectl rollout status deployment/java-app \
                      --timeout=300s
                '''
            }
        }

        // ==========================================
        // 9. Verify Deployment
        // ==========================================
        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment

                    echo "========== SERVICES =========="
                    kubectl get services
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
CI/CD PIPELINE COMPLETED SUCCESSFULLY
Application deployed to GKE
========================================
'''
        }

        failure {
            echo '''
========================================
CI/CD PIPELINE FAILED
Check the failed stage and logs
========================================
'''
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
        }
    }
}
