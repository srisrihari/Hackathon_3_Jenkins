
pipeline {
    agent any

    stages {
        stage('Build & Validate') {
            steps {
                echo 'Stage 1: Build & Validate'
                echo 'Validating environment...'
            }
        }

        stage('Deploy - Run ETL') {
            steps {
                echo 'Stage 2: Deploy - Run ETL'
                echo 'Running ETL process...'
            }
        }

        stage('Visualize') {
            steps {
                echo 'Stage 3: Visualize'
                echo 'Generating visualizations...'
            }
        }
    }

    post {
        success {
            echo 'Pipeline execution completed successfully'
        }
        failure {
            echo 'Pipeline execution failed'
        }
    }
}
