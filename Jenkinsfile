pipeline {
    agent any
    
    stages {
        stage('Build Backend Image') {
            steps {
                sh '''
                    echo "=== Building Docker Image ==="
                    docker build -t backend-app backend/
                    echo "=== Image built successfully ==="
                '''
            }
        }
        
        stage('Cleanup Old Containers') {
            steps {
                sh '''
                    echo "=== Force removing all old containers ==="
                    
                    # List all containers before cleanup
                    echo "Before cleanup:"
                    docker ps -a | grep -E "backend|nginx" || true
                    
                    # Remove by specific names
                    docker rm -f backend1 backend2 nginx-lb 2>/dev/null || true
                    
                    # Remove all containers with backend or nginx in name
                    docker ps -a | grep -E "backend|nginx" | awk '{print $1}' | xargs -r docker rm -f 2>/dev/null || true
                    
                    echo "After cleanup:"
                    docker ps -a | grep -E "backend|nginx" || echo "All clean! No containers found."
                '''
            }
        }
        
        stage('Deploy Backend Containers') {
            steps {
                sh '''
                    echo "=== Starting backend containers ==="
                    docker run -d --name backend1 backend-app
                    docker run -d --name backend2 backend-app
                    
                    echo "=== Running containers ==="
                    docker ps | grep backend
                '''
            }
        }
        
        stage('Deploy NGINX Load Balancer') {
            steps {
                sh '''
                    echo "=== Setting up NGINX Load Balancer ==="
                    
                    # Wait for backend containers to be ready
                    sleep 5
                    
                    echo "=== Using nginx config from repository ==="
                    cat nginx-lb.conf
                    
                    # Remove old nginx container if exists
                    docker rm -f nginx-lb 2>/dev/null || true
                    
                    # Start nginx container with config from current directory
                    docker run -d \
                        --name nginx-lb \
                        -p 80:80 \
                        -v $(pwd)/nginx-lb.conf:/etc/nginx/conf.d/default.conf \
                        --link backend1 \
                        --link backend2 \
                        nginx:alpine
                    
                    echo "=== NGINX Load Balancer deployed ==="
                    
                    # Show running containers
                    docker ps | grep nginx
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=== Verifying Deployment ==="
                    echo "Running containers:"
                    docker ps | grep -E "backend|nginx"
                    
                    echo "=== Testing NGINX Load Balancer ==="
                    sleep 5
                    curl -s -I http://localhost | head -n 1 || echo "Waiting for NGINX to start..."
                    
                    echo "=== Testing Backend Access ==="
                    sleep 2
                    curl -s http://localhost/app.cpp || echo "App endpoint not yet available"
                    
                    echo "=== Testing Load Balancing ==="
                    echo "First request:"
                    curl -s http://localhost | grep -o "backend[0-9]" || echo "Response doesn't show backend"
                    echo "Second request:"
                    curl -s http://localhost | grep -o "backend[0-9]" || echo "Response doesn't show backend"
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Pipeline Completed ==="
        }
        success {
            echo "✅ Pipeline executed successfully for PES1UG23AAM200!"
            sh '''
                echo "=== Final Running Containers ==="
                docker ps | grep -E "backend|nginx"
                
                echo "=== Testing Load Balancer ==="
                echo "Try these commands manually:"
                echo "curl http://localhost"
                echo "curl http://localhost/app.cpp"
                echo ""
                echo "Or open in browser: http://localhost"
            '''
        }
        failure {
            echo "❌ Pipeline failed. Check logs above for errors."
            sh '''
                echo "=== Debug Information ==="
                echo "Current directory: $(pwd)"
                echo "Docker images:"
                docker images | grep backend
                echo "Docker containers (all):"
                docker ps -a | grep -E "backend|nginx"
                echo "=== NGINX Config file ==="
                ls -la nginx-lb.conf 2>/dev/null || echo "No nginx config found"
                cat nginx-lb.conf 2>/dev/null || echo "Cannot read nginx config"
            '''
        }
        cleanup {
            echo "=== Cleanup Steps ==="
            // Uncomment to auto-cleanup after pipeline
            // sh '''
            //     echo "Cleaning up containers..."
            //     docker rm -f nginx-lb 2>/dev/null || true
            //     docker rm -f backend1 backend2 2>/dev/null || true
            // '''
        }
    }
}
