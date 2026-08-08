pipeline {

    agent any

    environment {

        IMAGE_NAME = "common-node-js"
        IMAGE_TAG  = "1.0.${BUILD_NUMBER}"

        PROJECT_ID = "healthcare-488108"
        REGION     = "us-east1"

        REPOSITORY = "raviregistry"

        IMAGE_URI = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${IMAGE_NAME}:${IMAGE_TAG}"


        // GCP Authentication
        GOOGLE_APPLICATION_CREDENTIALS = "C:/gcp/service-account.json"

        GCLOUD = "C:/Users/krred/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin/gcloud.cmd"


        // GKE Details
        CLUSTER_NAME = "ravi-cluster-1"

        CLUSTER_LOCATION = "us-central1"

        NAMESPACE = "default"

    }


    stages {


        stage('Checkout Code') {

            steps {

                git branch: 'master',
                url: 'https://github.com/kothakapurreddy/common-node-js.git'

            }
        }



        stage('Authenticate GCP') {

            steps {

                bat '''
                "%GCLOUD%" auth activate-service-account --key-file=%GOOGLE_APPLICATION_CREDENTIALS%

                "%GCLOUD%" config set project %PROJECT_ID%

                "%GCLOUD%" auth configure-docker %REGION%-docker.pkg.dev --quiet
                '''

            }
        }




        stage('Build Docker Image') {

            steps {

                bat '''
                docker build -t %IMAGE_URI% .
                '''

            }
        }




        stage('Push Docker Image') {

            steps {

                bat '''
                docker push %IMAGE_URI%
                '''

            }
        }





        stage('Connect To GKE Cluster') {

            steps {

                bat '''
                "%GCLOUD%" container clusters get-credentials %CLUSTER_NAME% --region %CLUSTER_LOCATION% --project %PROJECT_ID%
                '''

            }
        }





        stage('Create Kubernetes Deployment') {

            steps {

                bat '''

                kubectl apply -f deployment.yaml

                '''

            }

        }




        stage('Update Image') {

            steps {

                bat '''

                kubectl set image deployment/common-node-js \
                common-node-js=%IMAGE_URI% \
                -n %NAMESPACE%

                '''

            }

        }





        stage('Verify Deployment') {

            steps {

                bat '''

                kubectl get pods -n %NAMESPACE%

                kubectl get svc -n %NAMESPACE%

                '''

            }

        }


    }




    post {


        success {

            echo """
            ======================================
            DEPLOYMENT SUCCESSFUL

            Image:
            ${IMAGE_URI}

            Cluster:
            ${CLUSTER_NAME}

            ======================================
            """

        }


        failure {

            echo """
            ======================================
            DEPLOYMENT FAILED

            Check Jenkins Console Logs

            ======================================
            """

        }


        always {

            echo "Pipeline Completed"

        }

    }

}
