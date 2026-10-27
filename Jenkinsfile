pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'ankitkumawat2627/frontend-gitops'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.SKIP_BUILD = sh(
                        script: 'git log -1 --pretty=%s',
                        returnStdout: true
                    ).trim().contains('[skip ci]') ? 'true' : 'false'
                }
            }
        }

        stage('Build Image') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                sh 'docker build -t $DOCKER_IMAGE:$BUILD_NUMBER .'
            }
        }

        stage('Push Image') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "$DOCKER_IMAGE:$BUILD_NUMBER"
                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Manifest') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-repo-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        sed -i "s|image: .*|image: $DOCKER_IMAGE:$BUILD_NUMBER|" k8s/deployment.yaml
                        git config user.name "Jenkins"
                        git config user.email "jenkins@localhost"
                        git add k8s/deployment.yaml
                        git commit -m "Update frontend image to build $BUILD_NUMBER [skip ci]" || true
                        git -c http.extraheader="AUTHORIZATION: basic $(printf '%s:%s' "$GIT_USER" "$GIT_TOKEN" | base64 -w0)" push origin HEAD:main
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
    }
}
