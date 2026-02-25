pipeline {
    agent { label 'built-in' }
    options { skipDefaultCheckout() }
    stages {
        stage('Get Code') {
            steps {
                echo '---- CLEAN BEFORE BUILD STARTS ----'
                cleanWs()
                echo '---- DOWNLOAD REPO MASTER ----'
                checkout scmGit(
                    branches: [[name: '*/master']],
                    userRemoteConfigs: [[url: 'https://github.com/jguimeram/todo-list-aws']])
                echo '---- WORKSPACE ----'
                echo WORKSPACE
            }
        }

        stage('Set Environment') {
            steps {
                sh'''
                python3 -m venv .venv
                . .venv/bin/activate
                PYTHONPATH=$PWD
                pip install pytest requests
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh'''
                echo "---- DEPLOY PRODUCTION ----"
                sam build
                sam deploy \
                --template template.yaml \
                --config-env production \
                --stack-name production-todo-list-aws \
                --region us-east-1 \
                --resolve-s3 \
                --capabilities CAPABILITY_IAM \
                --parameter-overrides Stage="production" \
                --no-disable-rollback \
                --save-params \
                --no-confirm-changeset \
                --no-fail-on-empty-changeset
                '''
            }
        }

        stage('Rest Test') {
            steps {
                 catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                echo '---- REST TESTS ----'
                sh'''
                   python3 -m venv .venv
                   . .venv/bin/activate
                   export BASE_URL=$(aws cloudformation describe-stacks --stack-name production-todo-list-aws --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text)
                   echo $BASE_URL
                   pytest --junitxml=result-rest.xml -k "test_api_listtodos or test_api_gettodo" test/integration/todoApiTest.py
                   junit testResults: '**/result-rest.xml', stdioRetention: 'ALL'
                  '''
                 }
            }
        }
    }
       post {
        always {
             cleanWs()
        }
    }
}
