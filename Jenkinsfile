pipeline {
    agent any
    
    environment {
        FINAL_IMAGE = 'hamzachaieb01/ml-trained'
        DOCKER_TAG = 'latest'
        MLFLOW_DB = 'mlflow.db'
        EMAIL_TO = 'hitthetarget735@gmail.com'
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')  // Reduced timeout for running the MLflow server
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Docker Login') {
            steps {
                sh '''
                    echo "dckr_pat_CR7iXpPUQ_MegbA9oIIsyk4Jl5k" | docker login -u hamzachaieb01 --password-stdin
                '''
            }
        }
        
        stage('Pull MLflow Image') {
            steps {
                retry(3) {
                    sh 'docker pull ${FINAL_IMAGE}:${DOCKER_TAG}'
                }
            }
        }
        
        stage('Prepare MLflow Environment') {
            steps {
                sh '''
                    # Create mlflow.db if it doesn't exist
                    if [ ! -f ${MLFLOW_DB} ]; then
                        touch ${MLFLOW_DB}
                        echo "Created empty mlflow.db file"
                    fi
                    
                    # Check if MLflow is installed in the image; if not, install it dynamically
                    docker run --rm ${FINAL_IMAGE}:${DOCKER_TAG} pip show mlflow > /dev/null 2>&1 || {
                        echo "MLflow not found in image. Installing MLflow dynamically..."
                        docker run -d --name temp_mlflow_install ${FINAL_IMAGE}:${DOCKER_TAG} /bin/bash -c "pip install mlflow && touch /app/mlflow_installed"
                        docker cp temp_mlflow_install:/app/mlflow_installed .
                        docker rm -f temp_mlflow_install
                    }
                '''
            }
        }
        
        stage('Start MLflow Server on Port 5001') {
            steps {
                sh '''
                    # Remove any existing MLflow server container to avoid conflicts
                    docker rm -f mlflow_server || true
                    
                    # Run the MLflow server from the final image, mapping port 5001 on host to 5000 in container
                    # Use the image with MLflow (either pre-installed or dynamically added)
                    docker run -d --name mlflow_server -p 5001:5000 \
                    -v $(pwd)/${MLFLOW_DB}:/app/${MLFLOW_DB} \
                    ${FINAL_IMAGE}:${DOCKER_TAG} \
                    mlflow ui --backend-store-uri sqlite:///${MLFLOW_DB} --host 0.0.0.0 --port 5000 || {
                        echo "Warning: MLflow server failed to start. Check if MLflow and mlflow.db are configured correctly."
                        exit 1
                    }
                    
                    echo "MLflow server started. Access it at http://localhost:5001 to view metrics and visualizations for the XGBoost model"
                '''
            }
        }
    }
    
    post {
        success {
            emailext (
                subject: '$PROJECT_NAME - Build #$BUILD_NUMBER - SUCCESS',
                body: '''${SCRIPT, template="groovy-html.template"}
                
                Pipeline executed successfully!
                MLflow UI is available at: http://localhost:5001 to view metrics and visualizations for the XGBoost model.
                
                Check console output at $BUILD_URL to view the results.
                
                Changes:
                ${CHANGES}
                
                Failed Tests:
                ${FAILED_TESTS}
                ''',
                to: "${EMAIL_TO}",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                attachLog: true,
                compressLog: true
            )
            echo "✅ Pipeline executed successfully!"
        }
        failure {
            emailext (
                subject: '$PROJECT_NAME - Build #$BUILD_NUMBER - FAILURE',
                body: '''${SCRIPT, template="groovy-html.template"}
                
                Pipeline execution failed!
                
                Check console output at $BUILD_URL to view the results.
                
                Failed Stage: ${FAILED_STAGE}
                
                Error Message:
                ${BUILD_LOG}
                ''',
                to: "${EMAIL_TO}",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                attachLog: true,
                compressLog: true
            )
            echo "❌ Pipeline failed!"
        }
        always {
            echo "Pipeline execution complete!"
            sh '''
                docker rm -f mlflow_server temp_mlflow_install || true
                docker logout || true
                docker system prune -f || true
            '''
        }
    }
}
