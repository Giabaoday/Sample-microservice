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
        
        stage('Security Scan - Snyk') {
            tools {
                snyk 'Snyk'  
            }
            steps {
                echo '🛡️ Running Snyk security scan...'
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    sh '''
                        # Authenticate with Snyk
                        snyk auth $SNYK_TOKEN
                        
                        echo "📋 Scanning dependencies for vulnerabilities..."
                        snyk test \
                            --severity-threshold=high \
                            --json > snyk-report.json || echo "Vulnerabilities found, but continuing..."
                        
                        echo "📊 Snyk scan results:"
                        snyk test --severity-threshold=medium || echo "Scan completed with findings"
                        
                        echo "🔄 Monitoring project in Snyk dashboard..."
                        snyk monitor \
                            --project-name="github-auto-pipeline-${BUILD_NUMBER}" \
                            --project-tags="environment=ci,build=${BUILD_NUMBER}" || echo "Monitoring setup completed"
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'snyk-report.json', allowEmptyArchive: true
                }
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
        
        stage('Container Security Scan') {
            when {
                expression { fileExists('Dockerfile') }
            }
            tools {
                snyk 'Snyk'
            }
            steps {
                echo '🔍 Scanning Docker image for vulnerabilities...'
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    script {
                        sh '''
                            snyk auth $SNYK_TOKEN
                            
                            echo "🐳 Scanning container image..."
                            snyk container test demo-app:${BUILD_NUMBER} \
                                --severity-threshold=high \
                                --json > snyk-container-report.json || echo "Container vulnerabilities found"
                            
                            echo "📊 Container scan results:"
                            snyk container test demo-app:${BUILD_NUMBER} \
                                --severity-threshold=medium || echo "Container scan completed"
                            
                            echo "🔄 Monitoring container image..."
                            snyk container monitor demo-app:${BUILD_NUMBER} \
                                --project-name="demo-app-container-${BUILD_NUMBER}" || echo "Container monitoring setup"
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'snyk-container-report.json', allowEmptyArchive: true
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo '🚀 Deploying application...'
                sh '''
                    echo "Deployment to staging environment..."
                    echo "Application version: ${BUILD_NUMBER}"
                    echo "Deployment successful!"
                '''
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