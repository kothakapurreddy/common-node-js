pipeline {

    agent any

    environment {
        GCP_PROJECT = 'healthcare-488108'
        GCP_REGION  = 'us-central1'

        GAR_REPOSITORY = 'devops-repo'
        IMAGE_NAME     = 'java-app'

        GKE_CLUSTER = 'devops-gke'
        GKE_REGION  = 'us-central1'

        K8S_NAMESPACE = 'dev'

        GOOGLE_APPLICATION_CREDENTIALS =
            credentials('gcp-service-account')
    }

    stages {

        stage('Maven Build') {
            steps {
                sh '''
                    set -e
                    echo "Building Java application..."
                    mvn clean package -DskipTests
                    echo "Maven Build SUCCESS"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -e
                    echo "Building Docker image..."

                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    echo "Docker Build SUCCESS"
                '''
            }
        }

        stage('GCP Authentication') {
            steps {
                sh '''
                    set -e

                    gcloud auth activate-service-account \
                      --key-file="$GOOGLE_APPLICATION_CREDENTIALS"

                    gcloud config set project "$GCP_PROJECT"

                    echo "GCP Authentication SUCCESS"
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    set -e

                    gcloud auth configure-docker \
                      ${GCP_REGION}-docker.pkg.dev \
                      --quiet

                    IMAGE_URI="${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}"

                    docker tag \
                      ${IMAGE_NAME}:${BUILD_NUMBER} \
                      "$IMAGE_URI"

                    docker push "$IMAGE_URI"

                    echo "Docker Push SUCCESS"
                '''
            }
        }

        stage('GKE Authentication') {
            steps {
                sh '''
                    set -e

                    gcloud container clusters get-credentials \
                      "$GKE_CLUSTER" \
                      --region "$GKE_REGION" \
                      --project "$GCP_PROJECT"

                    echo "GKE Authentication SUCCESS"
                '''
            }
        }

        stage('Helm Deploy') {
            steps {
                sh '''
                    set -e

                    IMAGE_REPOSITORY="${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}"

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

                    echo "Helm Deployment SUCCESS"
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    kubectl get pods \
                      --namespace "$K8S_NAMESPACE"

                    kubectl get services \
                      --namespace "$K8S_NAMESPACE"

                    helm status java-app \
                      --namespace "$K8S_NAMESPACE"
                '''
            }
        }
    }

    post {

        success {
            echo "===================================="
            echo "PIPELINE SUCCESS"
            echo "Application deployed successfully"
            echo "===================================="
        }

        failure {
            echo "===================================="
            echo "PIPELINE FAILED"
            echo "Check failed stage"
            echo "===================================="
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
        }
    }
}
