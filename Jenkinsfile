pipeline {
    agent any 
    
    tools{
        jdk 'jdk11'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        DOCKER_IMAGE = "zacst/pet-clinic123:latest"
        CONTAINER_NAME = "petclinic-app"
        APP_PORT = "8083"
    }
    
    stages{
        
        stage("Git Checkout"){
            steps{
                git branch: 'main', changelog: false, poll: false, url: 'https://github.com/jaiswaladi246/Petclinic.git'
            }
        }
        
        stage("Compile"){
            steps{
                sh "mvn clean compile"
            }
        }
        
         stage("Test Cases"){
            steps{
                sh "mvn test"
            }
        }
        
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Petclinic \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=Petclinic '''
    
                }
            }
        }
        
        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format HTML ', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
         stage("Build"){
            steps{
                sh "mvn clean install"
            }
        }
        
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: '58be877c-9294-410e-98ee-6a959d73b352', toolName: 'docker') {
                        
                        sh "docker build -t ${DOCKER_IMAGE} ."
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }
        
        stage("TRIVY Security Scan"){
            steps{
                sh "trivy image ${DOCKER_IMAGE}"
            }
        }
        
        stage("Deploy with Docker"){
            steps{
                script{
                    sh '''
                        # Stop and remove existing container if it exists
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                        
                        # Pull latest image (optional, since we just built it)
                        docker pull ${DOCKER_IMAGE}
                        
                        # Run new container
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${APP_PORT}:8080 \
                            --restart unless-stopped \
                            ${DOCKER_IMAGE}
                        
                        # Verify container is running
                        docker ps -f name=${CONTAINER_NAME}
                        
                        # Wait for application to start
                        sleep 10
                        
                        # Health check
                        curl -f http://localhost:${APP_PORT}/petclinic || echo "Application may still be starting up"
                    '''
                }
            }
        }
        
        stage("Deployment Verification"){
            steps{
                script{
                    sh '''
                        # Check if container is running
                        if docker ps -f name=${CONTAINER_NAME} --format "table {{.Names}}" | grep -q ${CONTAINER_NAME}; then
                            echo "✅ Container ${CONTAINER_NAME} is running successfully"
                            docker logs --tail 10 ${CONTAINER_NAME}
                        else
                            echo "❌ Container ${CONTAINER_NAME} is not running"
                            exit 1
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            // Clean up unused Docker images to save space
            sh 'docker image prune -f || true'
        }
        success {
            echo "🎉 Pipeline completed successfully!"
            echo "Application is running at: http://localhost:${APP_PORT}/petclinic"
        }
        failure {
            echo "❌ Pipeline failed!"
            // Show container logs for debugging
            sh 'docker logs ${CONTAINER_NAME} || true'
        }
    }
}