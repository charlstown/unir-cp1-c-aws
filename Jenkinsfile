pipeline {
    agent any
    environment {
                STAGE = "staging"
                AWS_REGION = "us-east-1"
            }
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
        stage('Rest') {
            environment {
                PYTHONPATH="${WORKSPACE}"
            }
            steps {
                // Get API BASE_URL
            sh '''
            # Fetch the API Gateway URL from CloudFormation stack
            export BASE_URL=$(aws cloudformation describe-stacks \
                --stack-name todo-list-aws-${STAGE} \
                --region ${AWS_REGION} \
                --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                --output text)

            echo "API Gateway base url: $BASE_URL"

            BASE_URL=${BASE_URL} python3 -m pytest --junitxml=result-rest.xml test/integration/todoApiTest.py
            '''

            // Publish REST test results
            junit 'result-rest.xml'
            }
        }
        stage('Promote') {
            environment {
                GITHUB_TOKEN = credentials('GITHUB_TOKEN')  // Load GitHub Token
                GITHUB_USER = credentials('GITHUB_USER')  // Load GitHub User
                GITHUB_EMAIL = credentials('GITHUB_EMAIL')  // Load GitHub Email
            }
            steps {
                script {
                    sh '''

                    # Configuring Git
                    echo "Configuring Git"
                    git config --global user.name "$GITHUB_USER"
                    git config --global user.email "$GITHUB_EMAIL"
                    git remote set-url origin "https://x-access-token:$GITHUB_TOKEN@${GIT_URL#https://}"

                    # Tag the last stable commit as stable
                    git fetch --all --tags
                    LAST_COMMIT=$(git rev-parse HEAD)
                    echo "Tagging last stable commit: $LAST_COMMIT"
                    git tag -f $TAG_NAME $LAST_COMMIT
                    git push origin $TAG_NAME --force

                    # Ensure master is up to date
                    echo "Checking out master..."
                    git checkout master
                    git pull origin master

                    # Merge dev into master if no new commits exist in master
                    echo "Merging dev into master..."
                    git merge --ff-only dev || { echo "Merge failed! No fast-forward possible."; exit 1; }

                    # Push changes
                    echo "Pushing changes to master..."
                    git push origin master
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