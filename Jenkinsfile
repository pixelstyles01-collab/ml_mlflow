pipeline {
    agent any
    
    environment {
        FINAL_IMAGE = 'hamzachaieb01/ml-trained'
        DOCKER_TAG = 'latest'
        MLFLOW_DB = 'mlflow.db'
        EMAIL_TO = 'hitthetarget735@gmail.com'
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')  // Reduced timeout since we're only running the MLflow server
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
        
        stage('Start MLflow Server on Port 5001') {
            steps {
                sh '''
                    # Remove any existing MLflow server container to avoid conflicts
                    docker rm -f mlflow_server || true
                    
                    # Attempt to run the MLflow server from the final image, mapping port 5001 on host to 5000 in container
                    # Note: This may fail if MLflow is not installed in the image or if mlflow.db is missing
                    docker run -d --name mlflow_server -p 5001:5000 \
                    -v $(pwd)/${MLFLOW_DB}:/app/${MLFLOW_DB} \
                    ${FINAL_IMAGE}:${DOCKER_TAG} \
                    mlflow ui --backend-store-uri sqlite:///${MLFLOW_DB} --host 0.0.0.0 --port 5000 || {
                        echo "Warning: MLflow server failed to start. Check if MLflow and mlflow.db are present in the image."
                        exit 1
                    }
                    
                    echo "MLflow server started. Access it at http://localhost:5001 to view metrics and visualizations"
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
                docker rm -f mlflow_server || true
                docker logout || true
                docker system prune -f || true
            '''
        }
    }
}
