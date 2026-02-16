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
                echo '---- WHO AM I? ----'
                sh'''
                whoami
                '''
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
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            echo '---- STATIC ----'
                            sh'''
                            python3 -m venv .venv
                            . .venv/bin/activate
                            flake8 --exit-zero --format=pylint src > flake8.out
                            '''
                            recordIssues qualityGates: [[integerThreshold: 8, threshold: 8.0, type: 'TOTAL'], [criticality: 'FAILURE', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']], sourceCodeRetention: 'NEVER', tools: [flake8(pattern: 'flake8.out')]
                        }
                    }
                }

                stage('Security') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh'''
                            python3 -m venv .venv
                            . .venv/bin/activate
                            bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                            '''
                            recordIssues qualityGates: [[integerThreshold: 2, threshold: 2.0, type: 'TOTAL'], [criticality: 'FAILURE', integerThreshold: 4, threshold: 4.0, type: 'TOTAL']], sourceCodeRetention: 'NEVER', tools: [pyLint(pattern: 'bandit.out')]
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh'''
                ls -la
                sam build
                sam deploy --config-env staging \
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
            //BASE URL issue
            steps {
                sh'''
                   python3 -m venv .venv
                   . .venv/bin/activate
                   export BASE_URL=$(aws cloudformation describe-stacks --stack-name staging-todo-list-aws --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text)
                   echo $BASE_URL
                   pytest test/integration/todoApiTest.py
                  '''
            }
        }

        stage('Promote') {
            steps {
                withCredentials([string(credentialsId: 'c88df4f8-f1d2-4b25-bbe2-da9d8ac9a94e', variable: 'GITHUB')]) {
                    sh"""
                    # 1. Update remote URL
                    git config user.email "jenkins@yourdomain.com"
                    git config user.name "Jenkins CI"
                    git remote set-url origin https://jenkins:$GITHUB@github.com/jguimeram/todo-list-aws.git

                    # 2. Update TEST.md with Build Number
                    git checkout develop
                    git pull origin develop
                    date >> TEST.md
                    git add -A
                    git commit -m "Update changelog - Build #${BUILD_NUMBER}"

                    # 3. Merge to master
                    git fetch origin master
                    git checkout master
                    git pull origin master
                    git merge develop --no-edit
                    git tag -a "Release-${env.BUILD_NUMBER}" -m "Release version ${env.BUILD_NUMBER}"

                    # Push everything
                    git push origin develop master --tags

                    # Who am I?
                    whoami
                    hostname
                    """
                }
            }
        }
    }
        post {
        always {
            build job: 'todo-list-aws-cd-pipeline'
        }
        }
}
