pipeline {
    agent any
    stages {
        stage('Get Code') {
            steps {
                script {
                    // Print info
                    echo "[${STAGE_NAME}] Building branch ${GIT_BRANCH} of repository ${GIT_URL}"

                    // Add the workspace as a safe directory for Git
                    sh 'git config --global --add safe.directory ${WORKSPACE}'

                    // Clone the repository and check out the specified branch
                    checkout([$class: 'GitSCM',
                        branches: [[name: "${GIT_BRANCH}"]],
                        userRemoteConfigs: [[url: "${GIT_URL}"]]
                    ])
                }
            }
        }
        stage('Static Test') {
            environment {
                PYTHONPATH="${WORKSPACE}"
            }
            steps {
                // Run flake8 with pylint
                sh '''
                python3 -m flake8 --format=pylint --exit-zero --output-file=result-flake8.out src
                bandit -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}" || true
                '''
                // Publish Flake8 results (without quality gates)
                recordIssues(tools: [flake8(pattern: 'result-flake8.out')])

                // Publish Bandit security scan results (without quality gates)
                recordIssues(tools: [pyLint(name: 'bandit', pattern: 'bandit.out')])          
            }
        }
        stage('Deploy') {
            environment {
                STAGE = "staging"
                AWS_REGION = "us-east-1"
            }
            steps {
                script {
                    sh '''
                    # check aws identity
                    aws sts get-caller-identity

                    # Build the application
                    sam build --debug

                    # Validate the CloudFormation template
                    sam validate --region ${AWS_REGION}

                    # Deploy using the specified environment config
                    sam deploy --config-env ${STAGE} --region ${AWS_REGION} --no-confirm-changeset --no-fail-on-empty-changeset --debug
                    '''
                }
            }
        }
    }
    post {
        always {
            // Clean the workspace after the pipeline completes
            cleanWs()
        }
    }
}