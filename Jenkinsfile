pipeline {
    agent any
    options { skipDefaultCheckout() }
    stages {
        stage('Get Code') {
            steps {
                echo '---- CLEAN BEFORE BUILD STARTS ----'
                cleanWs()
                echo '---- DOWNLOAD REPO ----'
                checkout scm
                echo '---- WORKSPACE ----'
                echo WORKSPACE
                echo '---- WHO AM I? ----'
                sh'''
                whoami
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh'''
                sam build
                sam deploy \
                --stack-name staging-todo-list-aws \
                --resolve-s3 \
                --capabilities CAPABILITY_IAM \
                --no-confirm-changeset \
                --no-fail-on-empty-changeset
                '''
            }
        }
    }
}
