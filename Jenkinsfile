pipeline {
    agent any

    environment {
        GOOGLE_APPLICATION_CREDENTIALS = credentials('gcp-key')   // GCP service account key (Secret file)
        PROJECT_ID = 'nice-proposal-476909-g4'
        CLUSTER_NAME = 'jenkins-gke'
        CLUSTER_ZONE = 'us-central1-a'
        DOCKERHUB_CREDENTIALS = credentials('docker')            // Jenkins DockerHub creds (username/password)
        DOCKER_USERNAME = 'udayvarma825'                         // your DockerHub username
        IMAGE_TAG_SUFFIX = 'v1'                                  // change for versioning (v2, v3...)
    }

    stages {
        stage('Checkout') {
            steps {
                sh '''
                    rm -rf nginx-project
                    git clone https://github.com/udayvarma123/nginx-project.git
                '''
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                dir('nginx-project') {
                    sh '''
                        echo "Logging in to DockerHub..."
                        echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin

                        echo "Building and pushing BLUE image..."
                        docker build -t ${DOCKER_USERNAME}/nginx-blue:${IMAGE_TAG_SUFFIX} -f Dockerfile.blue .
                        docker push ${DOCKER_USERNAME}/nginx-blue:${IMAGE_TAG_SUFFIX}

                        echo "Building and pushing GREEN image..."
                        docker build -t ${DOCKER_USERNAME}/nginx-green:${IMAGE_TAG_SUFFIX} -f Dockerfile.green .
                        docker push ${DOCKER_USERNAME}/nginx-green:${IMAGE_TAG_SUFFIX}

                        docker logout
                    '''
                }
            }
        }

        stage('Authenticate to GCP') {
            steps {
                sh '''
                    echo "Authenticating to GCP..."
                    gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                    gcloud config set project $PROJECT_ID
                    gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $PROJECT_ID
                '''
            }
        }

        stage('Blue-Green Deployment') {
            steps {
                dir('nginx-project') {
                    sh '''
                        echo "Detecting current color..."
                        CURRENT_COLOR=$(kubectl get svc nginx-service -o=jsonpath='{.spec.selector.color}' 2>/dev/null || echo "none")
                        echo "Current active color: $CURRENT_COLOR"

                        if [ "$CURRENT_COLOR" = "blue" ]; then
                            NEW_COLOR="green"
                            DEPLOYMENT="nginx-deployment-green"
                            YAML_FILE="nginx-deployment-green.yaml"
                        else
                            NEW_COLOR="blue"
                            DEPLOYMENT="nginx-deployment-blue"
                            YAML_FILE="nginx-deployment-blue.yaml"
                        fi

                        IMAGE_FULL="${DOCKER_USERNAME}/nginx-${NEW_COLOR}:${IMAGE_TAG_SUFFIX}"
                        echo "New color: $NEW_COLOR"
                        echo "Will use image: $IMAGE_FULL"
                        echo "Using source YAML: $YAML_FILE"

                        # Create a temp copy so original in repo is not modified
                        TMP_YAML="/tmp/${YAML_FILE%%.yaml}-$BUILD_ID.yaml"
                        cp $YAML_FILE $TMP_YAML

                        # Replace the placeholder with the actual image name
                        sed -i "s|IMAGE_PLACEHOLDER|${IMAGE_FULL}|g" $TMP_YAML

                        echo "Applying YAML from $TMP_YAML"
                        kubectl apply -f $TMP_YAML

                        echo "Waiting for rollout of deployment/$DEPLOYMENT"
                        kubectl rollout status deployment/$DEPLOYMENT --timeout=5m
                        
                        if ! kubectl get svc nginx-service >/dev/null 2>&1; then
                  echo "Service not found — creating nginx-service..."
                  kubectl apply -f nginx-service.yaml
                fi

                        echo "Switching service to new color..."
                        kubectl patch svc nginx-service -p '{"spec": {"selector": {"app": "nginx", "color": "'$NEW_COLOR'"}}}'
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Verifying deployment..."
                    kubectl get pods -l app=nginx -o wide
                    kubectl get svc nginx-service
                '''
            }
        }

        stage('Cleanup Old Deployment') {
            steps {
                sh '''
                    echo "Cleaning old deployment if exists..."
                    # If there was an older color, delete its deployment to avoid duplicate pods
                    if [ "$CURRENT_COLOR" != "none" ]; then
                        echo "Deleting old deployment: nginx-deployment-$CURRENT_COLOR"
                        kubectl delete deployment nginx-deployment-$CURRENT_COLOR --ignore-not-found
                    else
                        echo "No previous deployment to delete."
                    fi
                '''
            }
        }
    }

    
}
