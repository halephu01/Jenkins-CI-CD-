pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        NETWORK_NAME = 'my-network'
    }

    stages {
        stage('Check Environment') {
            steps {
                script {
                    sh '''
                        echo "Workspace directory:"
                        pwd
                        echo "System information:"
                        uname -a
                        echo "Docker version:"
                        docker --version || echo "Docker not installed"
                        echo "Docker Compose version:"
                        docker-compose --version || echo "Docker Compose not installed"
                        echo "Node version:"
                        node -v || echo "Node.js not installed"
                        echo "NPM version:"
                        npm -v || echo "NPM not installed"
                    '''
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    try {
                        git branch:'main', url: 'https://github.com/halephu01/Jenkins-CI-CD.git'
                        sh 'ls -la'
                    } catch (Exception e) {
                        error("Clone Repository failed: ${e.getMessage()}")
                    }
                }
            }
        }

        stage('Docker Login') {
            steps {
                script {
                    try {
                        sh '''
                            echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                            echo "Docker login successful"
                        '''
                    } catch (Exception e) {
                        error("Docker login failed: ${e.getMessage()}")
                    }
                }
            }
        }

        stage('Prepare Prometheus Config') {
            steps {
                script {
                    sh '''
                        # Tạo thư mục nếu chưa tồn tại
                        mkdir -p docker/prometheus
                        
                        # Tạo file prometheus.yml
                        cat > docker/prometheus/prometheus.yml << 'EOL'
global:
  scrape_interval: 2s
  evaluation_interval: 2s

scrape_configs:
  - job_name: 'api-gateway'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.17.0.1:9000']
        labels:
          application: 'API Gateway'
  - job_name: 'product-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.17.0.1:8080']
        labels:
          application: 'Product Service'
  - job_name: 'order-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.17.0.1:8081']
        labels:
          application: 'Order Service'
  - job_name: 'inventory-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.17.0.1:8082']
        labels:
          application: 'Inventory Service'
  - job_name: 'notification-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.17.0.1:8083']
        labels:
          application: 'Notification Service'
  - job_name: 'identity-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.17.0.1:8087']
        labels:
          application: 'Identity Service'
EOL

                        # Cấp quyền cho file
                        chmod 644 docker/prometheus/prometheus.yml
                        ls -la docker/prometheus/
                        cat docker/prometheus/prometheus.yml
                    '''
                }
            }
        }

        stage('Run Docker Compose') {
            steps {
                script {
                    try {
                        sh '''
                            echo "Stopping and removing existing containers..."
                            docker-compose down || true
                            docker rm -f $(docker ps -aq) || true
                            docker network prune -f || true
                            
                            echo "Starting new containers..."
                            docker-compose up -d
                            
                            echo "Waiting for containers to start..."
                            sleep 30
                            
                            echo "Checking container status..."
                            docker-compose ps
                        '''
                    } catch (Exception e) {
                        echo "Error in Docker Compose stage: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                        error("Docker Compose stage failed")
                    }
                }
            }
        }

        stage('Pull and Run Services') {
            steps {
                script {
                    try {
                        sh '''
                            # Remove existing containers if any
                            docker rm -f api-gateway notification-service inventory-service order-service identity-service product-service || true
                            
                            # Pull images
                            docker pull 4miby/api-gateway
                            docker pull 4miby/notification-service
                            docker pull 4miby/inventory-service
                            docker pull 4miby/order-service
                            docker pull 4miby/identity-service
                            docker pull 4miby/product-service
                            
                            # Create network if not exists
                            docker network create app-network || true
                            
                            # Run containers
                            docker run -d --name api-gateway \
                                --network app-network \
                                -p 9000:9000 \
                                4miby/api-gateway

                            docker run -d --name notification-service \
                                --network app-network \
                                -p 8083:8083 \
                                4miby/notification-service

                            docker run -d --name inventory-service \
                                --network app-network \
                                -p 8082:8082 \
                                4miby/inventory-service

                            docker run -d --name order-service \
                                --network app-network \
                                -p 8081:8081 \
                                4miby/order-service

                            docker run -d --name identity-service \
                                --network app-network \
                                -p 8087:8087 \
                                4miby/identity-service

                            docker run -d --name product-service \
                                --network app-network \
                                -p 8080:8080 \
                                4miby/product-service
                            
                            # Verify containers are running
                            echo "Checking service status..."
                            docker ps --format "{{.Names}}: {{.Status}}"
                        '''
                    } catch (Exception e) {
                        echo "Error in Pull and Run Services stage: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                        error("Pull and Run Services stage failed")
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh 'docker ps'
                    sh 'docker-compose ps'
                }
            }
        }

        stage('Deploy Frontend') {
            steps {
                script {
                    sh '''
                        cd frontend
                        pwd
                        ls -la
                        npm -v
                        node -v
                        npm install
                        npm start &
                        sleep 10
                        curl -s http://localhost:3000 || echo "Frontend chưa sẵn sàng"
                    '''
                }
            }
        }

    }

    post {
        always {
            sh '''
                echo "Cleaning up..."
                docker logout || true
            '''
        }
        failure {
            script {
                sh '''
                    echo "Pipeline failed! Cleaning up..."
                    docker-compose down || true
                    docker rm -f $(docker ps -aq) || true
                    docker network prune -f || true
                '''
            }
        }
        success {
            echo "Pipeline completed successfully!"
        }
    }
}