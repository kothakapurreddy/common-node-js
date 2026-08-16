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
        GKE_REGION  = 'us-central1'

        // ==============================
        // Kubernetes / Helm
        // ==============================
        K8S_NAMESPACE = 'dev'

        // ==============================
        // Jenkins GCP Credential
        // ==============================
        GOOGLE_APPLICATION_CREDENTIALS =
            credentials('gcp-service-account')
    }

    stages {


        // ==========================================
        // 3. Maven Build
        // ==========================================
        stage('Maven Build') {
            steps {
                sh '''
                    set -e

                    echo "Building Java application..."

                    mvn clean package -DskipTests

                    echo "Maven build completed successfully."
                '''
            }
        }

        // ==========================================
        // 4. Docker Build
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
        // 5. GCP Authentication
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
        // 6. Docker Authentication
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
        // 7. Push Image to Artifact Registry
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
        // 8. Verify Artifact Registry
        // ==========================================
        stage('Verify Artifact Registry') {
            steps {
                sh '''
                    set -e

                    echo "Checking Artifact Registry..."

                    gcloud artifacts docker images list \
                      ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME} \
                      --include-tags
                '''
            }
        }

        // ==========================================
        // 9. GKE Authentication
        // ==========================================
        stage('GKE Authentication') {
            steps {
                sh '''
                    set -e

                    echo "Checking GKE cluster status..."

                    STATUS=$(gcloud container clusters describe \
                      "$GKE_CLUSTER" \
                      --region "$GKE_REGION" \
                      --project "$GCP_PROJECT" \
                      --format="value(status)")

                    echo "GKE Cluster Status: $STATUS"

                    if [ "$STATUS" != "RUNNING" ]; then
                        echo "ERROR: GKE cluster is not RUNNING."
                        exit 1
                    fi

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
        // 10. Helm Validate
        // ==========================================
        stage('Helm Validate') {
            steps {
                sh '''
                    set -e

                    echo "Validating Helm chart..."

                    helm lint ./helm/java-app

                    echo "Rendering Helm templates..."

                    helm template java-app ./helm/java-app \
                      --namespace "$K8S_NAMESPACE" \
                      --values ./helm/java-app/values-dev.yaml

                    echo "Helm validation completed."
                '''
            }
        }

        // ==========================================
        // 11. Deploy using Helm
        // ==========================================
        stage('Helm Deploy to GKE') {
            steps {
                sh '''
                    set -e

                    IMAGE_REPOSITORY="${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}"

                    echo "========================================"
                    echo "Deploying with Helm"
                    echo "========================================"

                    echo "Image Repository:"
                    echo "$IMAGE_REPOSITORY"

                    echo "Image Tag:"
                    echo "$BUILD_NUMBER"

                    echo "Namespace:"
                    echo "$K8S_NAMESPACE"

                    helm upgrade --install java-app \
                      ./helm/java-app \
                      --namespace "$K8S_NAMESPACE" \
                      --create-namespace \
                      --values ./helm/java-app/values-dev.yaml \
                      --set image.repository="$IMAGE_REPOSITORY" \
                      --set image.tag="$BUILD_NUMBER" \
                      --wait \
                      --timeout 5m \
                      --atomic

                    echo "Helm deployment completed successfully."
                '''
            }
        }

        // ==========================================
        // 12. Helm Status
        // ==========================================
        stage('Helm Status') {
            steps {
                sh '''
                    set -e

                    echo "========== HELM RELEASE =========="

                    helm status java-app \
                      --namespace "$K8S_NAMESPACE"

                    echo "========== HELM LIST =========="

                    helm list \
                      --namespace "$K8S_NAMESPACE"
                '''
            }
        }

        // ==========================================
        // 13. Verify Deployment
        // ==========================================
        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "========== PODS =========="

                    kubectl get pods \
                      --namespace "$K8S_NAMESPACE" \
                      -o wide

                    echo "========== DEPLOYMENT =========="

                    kubectl get deployment \
                      --namespace "$K8S_NAMESPACE"

                    echo "========== SERVICES =========="

                    kubectl get services \
                      --namespace "$K8S_NAMESPACE"

                    echo "========== HPA =========="

                    kubectl get hpa \
                      --namespace "$K8S_NAMESPACE" \
                      || true
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
CI/CD PIPELINE COMPLETED SUCCESSFULLY
========================================
Application deployed to GKE using Helm
'''
        }

        failure {
            echo '''
========================================
CI/CD PIPELINE FAILED
========================================
Check the failed Jenkins stage and logs
'''
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
            echo "GCP Project: ${GCP_PROJECT}"
            echo "GKE Cluster: ${GKE_CLUSTER}"
            echo "Kubernetes Namespace: ${K8S_NAMESPACE}"
        }
    }
}
