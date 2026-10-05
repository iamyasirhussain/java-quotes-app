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
        stage('Test') {
            steps {
                sh '''
                    if grep -qE '^[[:space:]]*$' quotes.txt; then
                        echo "FAIL: quotes.txt has blank lines"; exit 1
                    fi
                    if grep -q '"' quotes.txt; then
                        echo "FAIL: quotes.txt contains a double quote"; exit 1
                    fi
                    docker rm -f quotes-test 2>/dev/null || true
                    docker run -d --name quotes-test $IMAGE:$TAG
                    sleep 5
                    RESPONSE=$(docker exec quotes-test wget -qO- http://localhost:8000/) || { docker logs quotes-test; exit 1; }
                    echo "Response: $RESPONSE"
                    echo "$RESPONSE" | grep -Eq '^[{]"quote": ".+"[}]$' || { echo "FAIL: unexpected response"; exit 1; }
                '''
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
            sh 'docker rm -f quotes-test || true'
        }
    }
}
