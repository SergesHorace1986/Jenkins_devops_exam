pipeline {
    
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = 'DOCKER_HUB_PASS'
        GITHUB_CREDENTIALS    = 'github-creds'
        KUBECONFIG_CRED       = 'config'
        BRANCH = "${env.BRANCH_NAME ?: params.BRANCH ?: 'main'}"

        MOVIE_IMAGE = "horacio1986/movie-service"
        CAST_IMAGE  = "horacio1986/cast-service"
    }

    parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to deploy')
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

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/main"]],
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
                          docker build -t ${MOVIE_IMAGE}:${env.BRANCH}-${env.BUILD_NUMBER} movie-service/
                        """
                    }
                }

                stage('Build Cast Service') {
                    steps {
                        
                        echo "🔧 Building *Cast Service* image"

                        sh """
                          docker build -t ${CAST_IMAGE}:${env.BRANCH}-${env.BUILD_NUMBER} cast-service/
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
                      docker push ${MOVIE_IMAGE}:${env.BRANCH}-${env.BUILD_NUMBER}
                      docker push ${CAST_IMAGE}:${env.BRANCH}-${env.BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                
                echo "🚢 Starting *Helm Deployment* for branch ${env.BRANCH}"

                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'config')]) {
                    script {
                        // def helmFlags = "--atomic --timeout 5m0s"

                        if (env.BRANCH == "dev" || env.BRANCH == "main" || env.BRANCH.startsWith("feature/")) {
                            sh """
                              export KUBECONFIG=$config
                              helm upgrade --install  movie-platform-dev ./movie-platform \
                                -n dev \
                                -f movie-platform/values-dev.yaml \
                                --set movie_service.image.tag=main-13 \
                                --set cast_service.image.tag=main-13 \
                                --timeout 15m0s
                            """
                        }

                        if (env.BRANCH == "qa") {
                            sh """
                              export KUBECONFIG=$config
                              helm upgrade --install movie-platform-dev-qa ./movie-platform \
                                -n qa \
                                -f movie-platform/values-qa.yaml \
                                --set movie_service.image.tag=main-13 \
                                --set cast_service.image.tag=main-13 \
                                --timeout 15m0s
                            """
                        }

                        if (env.BRANCH == "staging") {
                            sh """
                              export KUBECONFIG=$config
                              helm upgrade --install movie-platform-staging ./movie-platform \
                                -n staging \
                                -f movie-platform/values-staging.yaml \
                                --set movie_service.image.tag=main-13 \
                                --set cast_service.image.tag=main-13 \
                                --timeout 15m0s
                            """
                        }

                        if (env.BRANCH == "main") {
                            timeout(time: 20, unit: 'MINUTES') {
                                input message: "Deploy to PRODUCTION?"
                            }
                            sh """
                              export KUBECONFIG=$config
                              helm upgrade --install movie-platform-prod ./movie-platform \
                                -n prod \
                                -f movie-platform/values-prod.yaml \
                                --set movie_service.image.tag=main-13 \
                                --set cast_service.image.tag=main-13 \
                                --timeout 15m0s
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            
            echo "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed from branch ${env.BRANCH}"    
        }
        failure {
            
            echo "❌ FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER} on branch ${env.BRANCH}"  
        }
        always {
            
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }
}
