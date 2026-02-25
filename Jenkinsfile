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
                echo ${env.WORKSPACE}
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

            stage('Promote') {
            agent { label 'built-in' }
            steps {
                unstash name: 'code'
                unstash name: 'updated_config'
                withCredentials([string(credentialsId: 'c88df4f8-f1d2-4b25-bbe2-da9d8ac9a94e', variable: 'GITHUB')]) {
                    sh"""
                    echo "---- UPDATE REMOTE URL ----"
                    git config user.email "jenkins@yourdomain.com"
                    git config user.name "Jenkins CI"
                    git remote set-url origin https://jenkins:$GITHUB@github.com/jguimeram/todo-list-aws.git

                    echo "---- ADD SAMCONFIG ----"
                    git checkout master
                    git fetch origin master
                    git add -A
                    git commit -m "add samconfig.toml"

                    echo "---- PUSH TO ORIGIN ----"
                    git push origin master

                    echo "---- WHO AM I? ----"
                    whoami
                    echo "---- HOSTNAME ----"
                    hostname
                    """
                }
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
                  '''
                   junit testResults: '**/result-rest.xml', stdioRetention: 'ALL'
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
