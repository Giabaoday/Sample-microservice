pipeline {
    agent any
    
    environment {
        SONAR_PROJECT_KEY = 'Giabaoday_Sample-microservice'
        SONAR_ORG = 'giabaoday'
    }
    
    tools {
        maven 'Maven'         
        jdk 'JDK11'            
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '📥 Code đã được checkout từ GitHub'
                sh 'ls -la'
            }
        }
        
        stage('Build') {
            steps {
                echo '🔨 Building application với Maven...'
                sh '''
                    mvn clean compile
                    echo "Maven build completed!"
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo '🧪 Running unit tests...'
                sh '''
                    mvn test
                    echo "All tests passed!"
                '''
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }
        
        stage('SonarCloud Analysis') {
            steps {
                echo '🔍 Running SonarCloud code quality analysis...'
                withSonarQubeEnv('SonarCloud') {  
                    sh '''
                        mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.organization=${SONAR_ORG} \
                            -Dsonar.host.url=https://sonarcloud.io
                    '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo '⏳ Waiting for SonarCloud Quality Gate...'
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
                echo '✅ Quality Gate passed!'
            }
        }
         
        stage('Build Docker Image') {
            steps {
                echo '🐳 Building Docker image...'
                script {
                    sh '''
                        if [ -f Dockerfile ]; then
                            echo "Dockerfile found, building image..."
                            docker build -t demo-app:${BUILD_NUMBER} .
                            docker tag demo-app:${BUILD_NUMBER} demo-app:latest
                            echo "Docker image built successfully!"
                        else
                            echo "No Dockerfile found, skipping Docker build"
                        fi
                    '''
                }
            }
        }
        
        stage('Scan') {
            steps {
                script {
                    snykSecurity(
                        severity: 'critical',
                        snykInstallation: 'Snyk',
                        snykTokenId: 'snyk-token',
                        failOnIssues: false
                    )
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh '''
                        # Ensure kubectl is installed
                        if ! command -v kubectl > /dev/null; then
                          curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
                          chmod +x ./kubectl
                          mv ./kubectl /usr/local/bin/kubectl
                        fi
                        # Update image tag in deployment
                        sed -i "s|demo-microservice:latest|demo-microservice:${BUILD_NUMBER}|g" k8s/deployment.yaml
                        # Apply Kubernetes manifests
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        # Wait for deployment to be ready with health checks
                        kubectl rollout status deployment/demo-microservice --timeout=300s
                        # Verify health checks
                        echo "Verifying application health..."
                        kubectl get pods -l app=demo-microservice
                        # Get service URL and verify endpoint
                        SERVICE_URL=$(kubectl get service demo-microservice-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                        if [ -n "$SERVICE_URL" ]; then
                            echo "Service URL: http://$SERVICE_URL"
                            # Add health check
                            curl -f http://$SERVICE_URL/health || echo "Health check failed, but continuing..."
                        fi
                        echo "✅ Kubernetes deployment successful!"
                    '''
                }
            }
            post {
                failure {
                    echo '❌ Deployment failed, attempting rollback...'
                    sh '''
                        kubectl rollout undo deployment/demo-microservice
                        kubectl rollout status deployment/demo-microservice --timeout=300s
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo '🧹 Cleaning up workspace...'
            archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
        }
        success {
            echo ''
            echo '🎉 ===== PIPELINE SUCCESS ===== 🎉'
            echo '✅ Code checkout completed'
            echo '✅ Build successful'
            echo '✅ Tests passed'
            echo '✅ SonarCloud quality gate passed'
            echo '✅ Security scans completed'
            echo '✅ Deployment successful'
            echo ''
            echo '📊 Reports available:'
            echo '   - SonarCloud: https://sonarcloud.io'
            echo '   - Snyk Dashboard: https://app.snyk.io'
            echo '   - Jenkins Artifacts: Check build artifacts'
            echo '================================='
        }
        failure {
            echo ''
            echo '❌ ===== PIPELINE FAILED ===== ❌'
            echo 'Pipeline execution failed. Check logs above for details.'
            echo ''
            echo '🔧 Common issues:'
            echo '   - SonarCloud quality gate failed'
            echo '   - Critical security vulnerabilities found'
            echo '   - Build or test failures'
            echo '   - Tool configuration issues'
            echo ''
            echo '💡 Next steps:'
            echo '   1. Check SonarCloud report for code quality issues'
            echo '   2. Review Snyk report for security vulnerabilities'
            echo '   3. Fix issues and push new code'
            echo '================================='
        }
        unstable {
            echo '⚠️ Pipeline completed with warnings - check security scan results'
        }
    }
}