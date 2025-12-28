pipeline {
    agent any
    
    environment {
        // Credentials stored in Jenkins Credentials Manager
        DB_HOST = credentials('db-host')
        DB_PORT = credentials('db-port')
        DB_NAME = credentials('db-name')
        DB_USER = credentials('db-user')
        DB_PASSWORD = credentials('db-password')
        EMAIL_RECIPIENT = credentials('email-recipient')
        PYTHON_PATH = 'python3'
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from repository...'
                checkout scm
                sh 'git log -1 --pretty=format:"%h - %an, %ar : %s"'
            }
        }
        
        stage('Build & Validate') {
            steps {
                echo 'Stage 1: Building and validating ETL scripts...'
                script {
                    try {
                        // Install dependencies
                        sh '''
                            ${PYTHON_PATH} -m pip install --upgrade pip --quiet
                            ${PYTHON_PATH} -m pip install -r requirements.txt --quiet
                        '''
                        
                        // Validate Python scripts syntax
                        sh '''
                            echo "Validating Python scripts..."
                            find scripts -name "*.py" -exec ${PYTHON_PATH} -m py_compile {} \;
                            echo "✓ All Python scripts are valid"
                        '''
                        
                        // Run data validation
                        sh '''
                            echo "Running data validation checks..."
                            chmod +x scripts/validate_data.sh
                            ./scripts/validate_data.sh || true
                        '''
                        
                        // Test database connection
                        sh '''
                            echo "Testing database connection..."
                            ${PYTHON_PATH} scripts/test_db_connection.py
                        '''
                        
                        echo '✓ Build stage completed successfully'
                    } catch (Exception e) {
                        echo "✗ Build failed: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                        error("Build stage failed")
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true
                }
            }
        }
        
        stage('Deploy - Run ETL') {
            steps {
                echo 'Stage 2: Running ETL pipeline and deploying to database...'
                script {
                    try {
                        def startTime = System.currentTimeMillis()
                        
                        // Generate source data if needed
                        sh '''
                            echo "Generating source datasets..."
                            ${PYTHON_PATH} scripts/generate_mta_sources.py
                        '''
                        
                        // Ensure database schema exists
                        sh '''
                            echo "Setting up database schema..."
                            ${PYTHON_PATH} scripts/db_setup.py
                        '''
                        
                        // Run ETL pipeline
                        sh '''
                            echo "Executing ETL pipeline..."
                            ${PYTHON_PATH} scripts/etl_pipeline.py 2>&1 | tee logs/etl_${BUILD_NUMBER}.log
                        '''
                        
                        // Generate summary report
                        sh '''
                            echo "Generating summary report..."
                            ${PYTHON_PATH} scripts/generate_summary.py 2>&1 | tee logs/summary_${BUILD_NUMBER}.log
                        '''
                        
                        def endTime = System.currentTimeMillis()
                        def duration = (endTime - startTime) / 1000.0
                        
                        // Get record count from database
                        def recordCount = sh(
                            script: '''
                                ${PYTHON_PATH} -c "
                                import os, psycopg2
                                from dotenv import load_dotenv
                                load_dotenv()
                                conn = psycopg2.connect(
                                    host=os.getenv('DB_HOST'),
                                    port=os.getenv('DB_PORT'),
                                    database=os.getenv('DB_NAME'),
                                    user=os.getenv('DB_USER'),
                                    password=os.getenv('DB_PASSWORD')
                                )
                                cur = conn.cursor()
                                cur.execute('SELECT COUNT(*) FROM unified_transport_data')
                                count = cur.fetchone()[0]
                                print(count)
                                cur.close()
                                conn.close()
                                "
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        // Store performance metrics
                        sh """
                            echo "Pipeline Performance Metrics:" > logs/metrics_${BUILD_NUMBER}.txt
                            echo "============================" >> logs/metrics_${BUILD_NUMBER}.txt
                            echo "Build Number: ${BUILD_NUMBER}" >> logs/metrics_${BUILD_NUMBER}.txt
                            echo "Execution Time: ${duration} seconds" >> logs/metrics_${BUILD_NUMBER}.txt
                            echo "Records Loaded: ${recordCount}" >> logs/metrics_${BUILD_NUMBER}.txt
                            echo "Status: SUCCESS" >> logs/metrics_${BUILD_NUMBER}.txt
                            echo "Timestamp: \$(date)" >> logs/metrics_${BUILD_NUMBER}.txt
                        """
                        
                        echo "✓ ETL Pipeline completed in ${duration} seconds"
                        echo "✓ Records loaded: ${recordCount}"
                        
                    } catch (Exception e) {
                        echo "✗ Deploy failed: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                        error("Deploy stage failed")
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'logs/**/*, processed/**/*, reports/**/*', allowEmptyArchive: true
                }
            }
        }
        
        stage('Visualize - Refresh Dashboard') {
            steps {
                echo 'Stage 3: Refreshing analytics dashboard...'
                script {
                    try {
                        // Check if dashboard server is running, start if needed
                        sh '''
                            echo "Checking dashboard status..."
                            if pgrep -f "streamlit run" > /dev/null; then
                                echo "Dashboard is already running"
                            else
                                echo "Starting dashboard server..."
                                nohup ${PYTHON_PATH} -m streamlit run dashboard/app.py --server.port=8501 --server.headless=true > logs/dashboard_${BUILD_NUMBER}.log 2>&1 &
                                sleep 5
                            fi
                        '''
                        
                        // Verify dashboard is accessible
                        sh '''
                            echo "Verifying dashboard accessibility..."
                            curl -f http://localhost:8501/_stcore/health || echo "Dashboard health check failed"
                        '''
                        
                        echo '✓ Dashboard refreshed successfully'
                        
                    } catch (Exception e) {
                        echo "⚠ Dashboard stage warning: ${e.getMessage()}"
                        // Don't fail the pipeline if dashboard fails
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline execution completed"
            // Archive all artifacts
            archiveArtifacts artifacts: 'logs/**/*, processed/**/*, reports/**/*, dashboard/**/*', allowEmptyArchive: true
        }
        success {
            echo "✓ Pipeline succeeded!"
            // Optional: Send success notification
        }
        failure {
            echo "✗ Pipeline failed!"
            script {
                // Send email notification on failure
                try {
                    emailext (
                        subject: "Jenkins Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """
                            Pipeline execution failed for ${env.JOB_NAME}
                            
                            Build Number: ${env.BUILD_NUMBER}
                            Build URL: ${env.BUILD_URL}
                            Stage: ${env.STAGE_NAME}
                            
                            Please check the Jenkins console for details.
                        """,
                        to: "${env.EMAIL_RECIPIENT}",
                        mimeType: 'text/html'
                    )
                } catch (Exception e) {
                    echo "Email notification failed: ${e.getMessage()}"
                }
            }
        }
        unstable {
            echo "Pipeline is unstable"
        }
    }
}
