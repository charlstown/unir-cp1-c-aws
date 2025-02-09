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
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'AWS_SESSION_TOKEN', variable: 'AWS_SESSION_TOKEN')
                ]) {
                    script {
                        sh '''
                        echo "Using AWS Region: $AWS_REGION"
                        echo "Using AWS Access Key: $AWS_ACCESS_KEY_ID"

                        # Export AWS credentials
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN}

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
    }
}