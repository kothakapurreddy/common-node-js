pipeline {

    agent any

    environment {

        // GCP
        GCP_PROJECT = 'healthcare-488108'
        GCP_REGION  = 'asia-south1'

        // Artifact Registry
        GAR_REPOSITORY = 'devops-repo'
        IMAGE_NAME     = 'java-app'

        // GKE
        GKE_CLUSTER = 'devops-gke'
        GKE_ZONE    = 'asia-south1-c'

        // Jenkins → GCP Service Account
        GOOGLE_APPLICATION_CREDENTIALS =
            credentials('service-account')
    }

    stages {
        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    echo "Docker image created:"
                    docker images | grep ${IMAGE_NAME}
                '''
            }
        }

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

        stage('Push Image to Artifact Registry') {
            steps {
                sh '''
                    echo "Tagging Docker image..."

                    docker tag \
                      ${IMAGE_NAME}:${BUILD_NUMBER} \
                      ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "Pushing image to Artifact Registry..."

                    docker push \
                      ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Verify Artifact Registry Image') {
            steps {
                sh '''
                    echo "Verifying image in Artifact Registry..."

                    gcloud artifacts docker images list \
                      ${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY} \
                      --include-tags
                '''
            }
        }

        stage('GKE Authentication') {
            steps {
                sh '''
                    echo "Authenticating with GKE..."

                    gcloud container clusters get-credentials \
                      "$GKE_CLUSTER" \
                      --zone "$GKE_ZONE" \
                      --project "$GCP_PROJECT"

                    echo "Checking GKE access..."

                    kubectl get nodes
                '''
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh '''
                    echo "Updating Kubernetes image..."

                    sed "s|IMAGE_PLACEHOLDER|${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPOSITORY}/${IMAGE_NAME}:${BUILD_NUMBER}|g" \
                      k8s/deployment.yaml > k8s/deployment-final.yaml

                    echo "Deploying application..."

                    kubectl apply -f k8s/deployment-final.yaml

                    if [ -f k8s/service.yaml ]; then
                        kubectl apply -f k8s/service.yaml
                    fi

                    echo "Waiting for deployment..."

                    kubectl rollout status deployment/java-app
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Pods:"
                    kubectl get pods -o wide

                    echo "Deployment:"
                    kubectl get deployment

                    echo "Services:"
                    kubectl get services
                '''
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo 'CI/CD PIPELINE SUCCESSFUL'
            echo 'Application deployed to GKE'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'CI/CD PIPELINE FAILED'
            echo 'Check the failed stage and Jenkins logs'
            echo '======================================'
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
        }
    }
}
