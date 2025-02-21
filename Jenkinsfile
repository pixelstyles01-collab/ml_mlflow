pipeline {
    agent any
    
    environment {
        FINAL_IMAGE = 'hamzachaieb01/ml-trained'
        DOCKER_TAG = 'latest'
        MLFLOW_DB = 'mlflow.db'
        EMAIL_TO = 'hitthetarget735@gmail.com'
        TEMP_IMAGE = 'hamzachaieb01/ml-trained-mlflow-temp'
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
                    
                    # Create a temporary image with MLflow installed, since the base image lacks it
                    echo "Creating temporary image with MLflow..."
                    docker run -d --name temp_mlflow_base ${FINAL_IMAGE}:${DOCKER_TAG} /bin/bash
                    docker commit temp_mlflow_base ${TEMP_IMAGE}:${DOCKER_TAG}
                    docker rm -f temp_mlflow_base
                    docker run -d --name temp_mlflow_install ${TEMP_IMAGE}:${DOCKER_TAG} /bin/bash -c "pip install mlflow && touch /app/mlflow_installed"
                    docker commit temp_mlflow_install ${TEMP_IMAGE}:${DOCKER_TAG}
                    docker rm -f temp_mlflow_install
                '''
            }
        }
        
        stage('Log Metrics to mlflow.db') {
            steps {
                sh '''
                    # Use the temporary image to log metrics to mlflow.db (you'll need to provide or generate this data)
                    echo "Logging metrics to mlflow.db..."
                    docker run --rm -v $(pwd)/${MLFLOW_DB}:/app/${MLFLOW_DB} ${TEMP_IMAGE}:${DOCKER_TAG} \
                    python -c "
                    import mlflow
                    from xgboost import XGBClassifier
                    import joblib
                    import os

                    # Set MLflow tracking URI
                    mlflow.set_tracking_uri('sqlite:///${MLFLOW_DB}')

                    # Load the model from the image (assuming it's saved as model.pkl)
                    model = joblib.load('/app/model.pkl')

                    # Start a run and log dummy metrics (replace with actual metrics from your model)
                    with mlflow.start_run():
                        mlflow.log_params({
                            'objective': 'binary:logistic',
                            'max_depth': 6,
                            'learning_rate': 0.1,
                            'n_estimators': 100,
                            'random_state': 42,
                            'min_child_weight': 1,
                            'subsample': 0.8,
                            'colsample_bytree': 0.8,
                        })
                        mlflow.log_metrics({
                            'accuracy': 0.9505,  # Replace with actual model accuracy
                            'precision_0': 0.95,
                            'recall_0': 0.95,
                            'f1_0': 0.95,
                            'precision_1': 0.95,
                            'recall_1': 0.95,
                            'f1_1': 0.95,
                        })
                        mlflow.xgboost.log_model(model, 'xgboost_model')
                    "
                '''
            }
        }
        
        stage('Start MLflow Server on Port 5001') {
            steps {
                sh '''
                    # Remove any existing MLflow server container to avoid conflicts
                    docker rm -f mlflow_server || true
                    
                    # Run the MLflow server from the temporary image, mapping port 5001 on host to 5000 in container
                    docker run -d --name mlflow_server -p 5001:5000 \
                    -v $(pwd)/${MLFLOW_DB}:/app/${MLFLOW_DB} \
                    ${TEMP_IMAGE}:${DOCKER_TAG} \
                    mlflow ui --backend-store-uri sqlite:///${MLFLOW_DB} --host 0.0.0.0 --port 5000 || {
                        echo "Warning: MLflow server failed to start. Check if MLflow and mlflow.db are configured correctly."
                        exit 1
                    }
                    
                    echo "MLflow server started. Access it at http://localhost:5001 to view metrics and visualizations for the XGBoost model"
                '''
            }
        }
        
        stage('Cleanup Temporary Image') {
            steps {
                sh '''
                    # Remove the temporary image
                    docker rmi -f ${TEMP_IMAGE}:${DOCKER_TAG} || true
                    rm -f mlflow_installed || true
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
                docker rm -f mlflow_server temp_mlflow_base temp_mlflow_install || true
                docker logout || true
                docker system prune -f || true
            '''
        }
    }
}
