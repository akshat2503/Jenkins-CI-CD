pipeline {
    agent any

    environment {
        PROJECT_ID = 'PROJECT_ID'
        CLUSTER_NAME = 'CLUSTER_NAME'
        CLUSTER_ZONE = 'CLUSTER_ZONE'
    }

    stages {
        stage('Authenticate with Google Cloud') {
            steps {
                withCredentials([file(credentialsId: 'service-acc-cred', variable: 'GC_KEY')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file="$GC_KEY"
                        gcloud config set project $PROJECT_ID
                        gcloud auth configure-docker gcr.io -q
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
                sh 'docker push gcr.io/$PROJECT_ID/portfolio-website:latest'
            }
        }

        stage('Deploy to GKE') {
            steps {
                withCredentials([file(credentialsId: 'service-acc-cred', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
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
