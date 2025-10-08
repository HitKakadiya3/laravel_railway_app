pipeline {
    agent any

    environment {
        IMAGE_NAME = "hitendra369/laravel_railway_app"
        TAG = "${BUILD_NUMBER}"
        DOCKERHUB_USER = credentials('DOCKERHUB_USERNAME')
        DOCKERHUB_PASS = credentials('DOCKERHUB_PASSWORD')
        RAILWAY_TOKEN = credentials('RAILWAY_TOKEN')
        RAILWAY_SERVICE_ID = credentials('RAILWAY_SERVICE_ID')
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build Laravel') {
            steps {
                sh '''
                    composer install --no-interaction --prefer-dist --optimize-autoloader
                    php artisan key:generate
                '''
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                sh '''
                    # Debug: Check if credentials are available
                    echo "Docker Hub User: $DOCKERHUB_USER"
                    echo "Password length: ${#DOCKERHUB_PASS}"
                    
                    # Login to Docker Hub
                    echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                    
                    if [ $? -ne 0 ]; then
                        echo "❌ Docker login failed!"
                        exit 1
                    fi
                    
                    echo "✅ Docker login successful"

                    # Build Docker image
                    docker build -t ${IMAGE_NAME}:${TAG} .
                    docker tag ${IMAGE_NAME}:${TAG} ${IMAGE_NAME}:latest

                    # Push Docker images
                    docker push ${IMAGE_NAME}:${TAG}
                    docker push ${IMAGE_NAME}:latest
                    
                    echo "✅ Docker images pushed successfully"
                '''
            }
        }

        stage('Deploy to Railway') {
            steps {
                sh '''
                    curl -sSL https://raw.githubusercontent.com/railwayapp/cli/master/install.sh | sh
                    export PATH="$HOME/.railway/bin:$PATH"
                    railway login --apiKey "$RAILWAY_TOKEN"

                    # Redeploy latest Docker image
                    railway redeploy --service "$RAILWAY_SERVICE_ID" -y
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            echo "✅ Deployment pipeline finished for build ${BUILD_NUMBER}"
        }
        failure {
            echo "❌ Pipeline failed! Check the logs above for details."
        }
        success {
            echo "🎉 Pipeline completed successfully !"
        }
    }
}
