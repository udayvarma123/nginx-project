pipeline {
    agent any

    environment {
        GOOGLE_APPLICATION_CREDENTIALS = credentials('gcp-key') // Jenkins secret JSON key
        PROJECT_ID = 'nice-proposal-476909-g4'
        CLUSTER_NAME = 'jenkins-gke'
        CLUSTER_ZONE = 'us-central1-a'
    }

    stages {
        stage('Checkout') {
            steps {
               sh 'rm -rf nginx-project'
                sh 'git clone https://github.com/udayvarma123/nginx-project.git'
            }
        }
        stage('Authenticate to GCP') {
            steps {
                sh '''
                    gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                    gcloud config set project $PROJECT_ID
                    gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $PROJECT_ID
                '''
            }
        }
   stage('Deploy to GKE(Blue-Green)') {
            steps {
                dir('nginx-project') {
                    sh '''
                        # Identify which color is active
                        CURRENT_COLOR=$(kubectl get svc nginx-service -o=jsonpath='{.spec.selector.color}' 2>/dev/null || echo "none")
                        echo "Current active color: $CURRENT_COLOR"

                        if [ "$CURRENT_COLOR" = "blue" ]; then
                            NEW_COLOR="green"
                            DEPLOYMENT="nginx-deployment-green"
                        else
                            NEW_COLOR="blue"
                            DEPLOYMENT="nginx-deployment-blue"
                        fi

                        echo "Deploying new color: $NEW_COLOR"

                        kubectl apply -f nginx-deployment-$NEW_COLOR.yaml
                        kubectl rollout status deployment/$DEPLOYMENT

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
                    kubectl get hpa
                '''
            }
        }
        }
}
