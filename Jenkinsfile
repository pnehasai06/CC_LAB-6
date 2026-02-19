pipeline {
    agent any
    
    stages {
        stage('Build Backend Image') {
            steps {
                sh '''
                    echo "Current directory: $(pwd)"
                    echo "Listing files: $(ls -la)"
                    docker build -t backend-app ./backend
                '''
            }
        }
        
        stage('Deploy Backend Containers') {
            steps {
                sh '''
                    docker rm -f backend1 backend2 2>/dev/null || true
                    docker run -d --name backend1 backend-app
                    docker run -d --name backend2 backend-app
                    echo "Backend containers deployed:"
                    docker ps | grep backend
                '''
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed. Check logs for details."
        }
        failure {
            echo "Pipeline failed. Check console logs for errors."
        }
    }
}
