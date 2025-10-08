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
                script {
                    try {
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
                            if [ $? -ne 0 ]; then
                                echo "❌ Docker build failed!"
                                exit 1
                            fi
                            
                            docker tag ${IMAGE_NAME}:${TAG} ${IMAGE_NAME}:latest

                            # Push Docker images
                            docker push ${IMAGE_NAME}:${TAG}
                            if [ $? -ne 0 ]; then
                                echo "❌ Docker push failed for tag ${TAG}!"
                                exit 1
                            fi
                            
                            docker push ${IMAGE_NAME}:latest
                            if [ $? -ne 0 ]; then
                                echo "❌ Docker push failed for latest tag!"
                                exit 1
                            fi
                            
                            echo "✅ Docker images pushed successfully"
                        '''
                    } catch (Exception e) {
                        error "Docker build and push stage failed: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Deploy to Railway') {
            steps {
                script {
                    try {
                        sh '''
                            curl -sSL https://raw.githubusercontent.com/railwayapp/cli/master/install.sh | sh
                            if [ $? -ne 0 ]; then
                                echo "❌ Railway CLI installation failed!"
                                exit 1
                            fi
                            
                            export PATH="$HOME/.railway/bin:$PATH"
                            railway login --apiKey "$RAILWAY_TOKEN"
                            if [ $? -ne 0 ]; then
                                echo "❌ Railway login failed!"
                                exit 1
                            fi

                            # Redeploy latest Docker image
                            railway redeploy --service "$RAILWAY_SERVICE_ID" -y
                            if [ $? -ne 0 ]; then
                                echo "❌ Railway deployment failed !"
                                exit 1
                            fi
                            
                            echo "✅ Railway deployment successful"
                        '''
                    } catch (Exception e) {
                        error "Railway deployment stage failed: ${e.getMessage()}"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                sh '''
                    echo "🧹 Cleaning up..."
                    
                    # Check Docker daemon accessibility
                    if docker info >/dev/null 2>&1; then
                        echo "✅ Docker daemon accessible"
                        docker logout || echo "⚠️ Docker logout failed or already logged out"
                        
                        # Try Docker cleanup with permission check
                        if docker system df >/dev/null 2>&1; then
                            docker system prune -f || echo "⚠️ Docker cleanup failed - continuing anyway"
                        else
                            echo "⚠️ Insufficient permissions for Docker cleanup"
                        fi
                    else
                        echo "⚠️ Docker daemon not accessible - skipping Docker cleanup"
                        echo "This is normal in some Jenkins environments"
                    fi
                    
                    # Alternative cleanup - remove build artifacts
                    rm -rf vendor/ node_modules/ || echo "⚠️ Artifact cleanup failed"
                    echo "✅ Basic cleanup completed"
                '''
                echo "✅ Deployment pipeline finished for build ${BUILD_NUMBER}"
            }
        }
        failure {
            echo "❌ Pipeline failed! Check the logs above for details."
            script {
                sh '''
                    echo "📊 Failure diagnostics:"
                    docker --version || echo "Docker not available"
                    echo "Available space: $(df -h / 2>/dev/null || echo 'Unable to check disk space')"
                    which railway || echo "Railway CLI not found"
                    echo "Jenkins user: $(whoami)"
                    echo "Docker socket permissions: $(ls -la /var/run/docker.sock 2>/dev/null || echo 'Docker socket not accessible')"
                '''
            }
        }
        success {
            echo "🎉 Pipeline completed successfully! Application deployed to Railway."
            script {
                sh '''
                    echo "📈 Success summary:"
                    echo "✅ Docker image: ${IMAGE_NAME}:${TAG}"
                    echo "✅ Service ID: ${RAILWAY_SERVICE_ID}"
                    echo "✅ Build number: ${BUILD_NUMBER}"
                '''
            }
        }
    }
}
