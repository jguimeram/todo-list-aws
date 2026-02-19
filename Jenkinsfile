pipeline {
    agent  { label 'built-in' }
    options { skipDefaultCheckout() }
    stages {
        stage('Get Code') {
            steps {
                echo '---- CLEAN BEFORE BUILD STARTS ----'
                cleanWs()
                echo '---- DOWNLOAD REPO ----'
                checkout scmGit(
                branches: [[name: '*/develop']],
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
                pip install requests pytest flake8 bandit
                '''
            }
        }

        stage('Static Tests')
        {
            parallel
            {
                stage('Flake8') {
                    steps {
                            echo '---- STATIC ----'
                            sh'''
                            python3 -m venv .venv
                            . .venv/bin/activate
                            flake8 --exit-zero --format=pylint src > flake8.out
                            '''     
                    }
                    post {
                        always{
                        recordIssues qualityGates: [], sourceCodeRetention: 'NEVER', tools: [flake8(pattern: 'flake8.out')]   
                        }
                    }
                }

                stage('Security') {
                    steps {
                            sh'''
                            python3 -m venv .venv
                            . .venv/bin/activate
                            bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                            '''
                    }
                    post{
                        always{
                             recordIssues qualityGates: [], sourceCodeRetention: 'NEVER', tools: [pyLint(pattern: 'bandit.out')]
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh'''
                echo "---- DEPLOY ----"
                sam build
                sam deploy \
                --template template.yaml \
                --config-env staging \
                --stack-name staging-todo-list-aws \
                --region us-east-1 \
                --resolve-s3 \
                --capabilities CAPABILITY_IAM \
                --no-disable-rollback \
                --parameter-overrides Stage="staging" \
                --save-params \
                --no-confirm-changeset \
                --no-fail-on-empty-changeset \
                '''
            }
        }

        stage('Rest Test') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                sh'''
                   python3 -m venv .venv
                   . .venv/bin/activate
                   export BASE_URL=$(aws cloudformation describe-stacks --stack-name staging-todo-list-aws --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text)
                   echo $BASE_URL
                   pytest test/integration/todoApiTest.py
                  '''
                }
            }
        }

          stage('Promote') {
            steps {
                withCredentials([string(credentialsId: 'c88df4f8-f1d2-4b25-bbe2-da9d8ac9a94e', variable: 'GITHUB')]) {
                    sh"""
                    echo "---- UPDATE REMOTE URL ----"
                    git config user.email "jenkins@yourdomain.com"
                    git config user.name "Jenkins CI"
                    git remote set-url origin https://jenkins:$GITHUB@github.com/jguimeram/todo-list-aws.git

                    echo "---- UPDATE CHANGELOG ----"
                    git checkout develop
                    git pull origin develop
                    echo "Build #${BUILD_NUMBER}-R1" >> CHANGELOG.md
                    git add -A
                    git commit -m "Update changelog - Build #${BUILD_NUMBER}-R1"

                    echo "---- MERGE TO MASTER ----"
                    git checkout master
                    git pull origin master
                    git merge develop --no-edit
                    git tag -a "Release-${env.BUILD_NUMBER}R1" -m "Release version ${env.BUILD_NUMBER}-R1"

                    echo "--- PUSH ----"
                    git push origin develop master --tags
                    """
                }
            }
        }
    }
        
    post {
        success {
            build job: 'todo-list-aws-cd-pipeline'
        }
    }
}