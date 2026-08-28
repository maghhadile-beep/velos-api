pipeline {
    agent any

    environment {
        IMAGE = "maghmhad/velos-api"
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Tester') {
            steps {
                script {
                    env.TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                }
                sh 'docker build --target test -t velos-api:test-$TAG .'
            }
        }

        stage('Construire') {
            steps {
                sh 'docker build -t $IMAGE:$TAG -t $IMAGE:latest .'
            }
        }

        stage('Publier') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'REGISTRE_USER',
                    passwordVariable: 'REGISTRE_PASS'
                )]) {
                    sh '''
                        echo "$REGISTRE_PASS" | docker login -u "$REGISTRE_USER" --password-stdin
                        docker push $IMAGE:$TAG
                        docker push $IMAGE:latest
                    '''
                }
            }
        }

        stage('Deployer') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-velos',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        kubectl set image deployment/velos-api api=$IMAGE:$TAG
                        kubectl rollout status deployment/velos-api --timeout=180s
                        kubectl get pods -l app=velos-api
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "OK : ${env.IMAGE}:${env.TAG} est deploye."
        }
        failure {
            echo "ECHEC : aucune publication ni deploiement si les tests ont echoue."
        }
        always {
            sh 'docker logout || true'
            sh 'docker image prune -f || true'
        }
    }
}
