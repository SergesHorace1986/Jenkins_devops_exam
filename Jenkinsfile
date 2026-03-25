pipeline {
    
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = 'DOCKER_HUB_PASS'
        GITHUB_CREDENTIALS    = 'github-creds'
        KUBECONFIG_CRED       = 'config'

        MOVIE_IMAGE = "horacio1986/movie-service"
        CAST_IMAGE  = "horacio1986/cast-service"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }
        stage('Build Started') {
            steps {
                echo "📥 Starting Checkout stage for ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            }
        }

        stage('Checkout') {
            steps {    
                
                echo "📥 Starting Checkout stage for ${env.JOB_NAME} #${env.BUILD_NUMBER}"

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/SergesHorace1986/Jenkins_devops_exam.git',
                        credentialsId: "${GITHUB_CREDENTIALS}"
                    ]]
                ])
            }
        }

        stage('Build Images') {
            parallel {

                stage('Build Movie Service') {
                    steps {

                        echo "🔧 Building *Movie Service* image"

                        sh """
                          docker build -t ${MOVIE_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} movie-service/
                        """
                    }
                }

                stage('Build Cast Service') {
                    steps {
                        
                        echo "🔧 Building *Cast Service* image"

                        sh """
                          docker build -t ${CAST_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} cast-service/
                        """
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                
                echo "📤 Pushing Docker images to DockerHub"

                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push ${MOVIE_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER}
                      docker push ${CAST_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                
                echo "🚢 Starting *Helm Deployment* for branch ${env.BRANCH_NAME}"

                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'config')]) {
                    script {
                        def helmFlags = "--atomic --timeout 5m0s"

                        if (env.BRANCH_NAME == "dev" || env.BRANCH_NAME == "main" || env.BRANCH_NAME.startsWith("feature/")) {
                            sh """
                              export KUBECONFIG=${config}
                              helm upgrade --install movie-platform-dev ./movie-platform \
                                -n dev \
                                -f movie-platform/values-dev.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (env.BRANCH_NAME == "qa") {
                            sh """
                              export KUBECONFIG=${config}
                              helm upgrade --install movie-platform-qa ./movie-platform \
                                -n qa \
                                -f movie-platform/values-qa.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (env.BRANCH_NAME == "staging") {
                            sh """
                              export KUBECONFIG=${config}
                              helm upgrade --install movie-platform-staging ./movie-platform \
                                -n staging \
                                -f movie-platform/values-staging.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (env.BRANCH_NAME == "main") {
                            timeout(time: 20, unit: 'MINUTES') {
                                input message: "Deploy to PRODUCTION?"
                            }
                            sh """
                              export KUBECONFIG=${config}
                              helm upgrade --install movie-platform-prod ./movie-platform \
                                -n prod \
                                -f movie-platform/values-prod.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            
            echo "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed from branch ${env.BRANCH_NAME}"    
        }
        failure {
            
            echo "❌ FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER} on branch ${env.BRANCH_NAME}"  
        }
        always {
            
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }
}
