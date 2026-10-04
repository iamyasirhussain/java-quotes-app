pipeline {
    agent any
    environment {
        IMAGE = "hussainy/java-quotes-app"
        TAG   = "${BUILD_NUMBER}"
    }
    stages {
        stage('Build image') {
            steps {
                sh 'docker build -t $IMAGE:$TAG .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$TAG
                        docker logout
                    '''
                }
            }
        }
        stage('Update k8s manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-k8s-creds', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        rm -rf k8s-repo
                        git clone https://$GH_USER:$GH_TOKEN@github.com/iamyasirhussain/java-quotes-k8s.git k8s-repo
                        cd k8s-repo
                        sed -i "s|image: .*|image: $IMAGE:$TAG|" app/deployment.yaml
                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@local"
                        git add app/deployment.yaml
                        git commit -m "Update image to $IMAGE:$TAG"
                        git push
                    '''
                }
            }
        }
    }
    post {
        always {
            sh 'rm -rf k8s-repo'
        }
    }
}
