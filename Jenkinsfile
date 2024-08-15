pipeline {
    agent any

    environment {
        PROJECT_ID = 'imposing-pager-432316-a8'
        CLUSTER_NAME = 'cluster-test'
        CLUSTER_ZONE = 'asia-northeast1-a'
    }

    stages {
        stage('Authenticate with Google Cloud') {
            steps {
                withCredentials([file(credentialsId: 'serviceAccKey', variable: 'GC_KEY')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file="$GC_KEY"
                        gcloud config set project $PROJECT_ID
                        gcloud auth configure-docker gcr.io -q
                        export PATH=$PATH:$HOME/bin
                    '''
                }
            }
        }
        
        stage('Clean Workspace') {
            steps {
                deleteDir() // Deletes the current workspace content
            }
        }
        
        stage('Checkout') {
            steps {
                sh 'git clone https://github.com/akshat2503/akshatgautam'
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'cd akshatgautam && docker build -t gcr.io/$PROJECT_ID/portfolio-website:$BUILD_NUMBER .'
                sh 'cd akshatgautam && docker build -t gcr.io/$PROJECT_ID/portfolio-website:latest .'
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh 'docker pull gcr.io/$PROJECT_ID/portfolio-website:latest'
                sh "trivy image --format table -o trivy-report.html gcr.io/$PROJECT_ID/portfolio-website:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push gcr.io/$PROJECT_ID/portfolio-website:$BUILD_NUMBER'
                sh 'docker push gcr.io/$PROJECT_ID/portfolio-website:latest'
            }
        }

        stage('Deploy to GKE') {
            steps {
                withCredentials([file(credentialsId: 'serviceAccKey', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh """
                        export PATH=$PATH:$HOME/bin
                        gcloud config set project "$PROJECT_ID"
                        gcloud container clusters get-credentials "$CLUSTER_NAME" --zone "$CLUSTER_ZONE"
                        kubectl apply -f akshatgautam/k8s/deployment.yaml
                    """
                }
            }
        }
        
        stage('Verify Deployment'){
            steps {
                sh 'kubectl get services'
            }
        }
    }
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
    
                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """
    
                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'ronitakki3@gmail.com',
                    from: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html,trivy-report.html'
                )
            }
        }
    }
}
