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
                echo whoami
                '''
            }
        }

        stage('Set Environment') {
            steps {
                sh'''
                python3 -m venv .venv
                . .venv/bin/activate
                PYTHONPATH=$PWD
                pip install flake8 bandit requests pytest
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

        stage('Rest Test') {
            //BASE URL issue
            steps {
                sh'''
                   python3 -m venv .venv
                   . .venv/bin/activate
                   pytest test/integration/todoApiTest.py
                  '''
            }
        }

        stage('Promote') {
            steps {
                echo 'promote'
            }
        }
    }
}
