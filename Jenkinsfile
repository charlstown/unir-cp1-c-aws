pipeline {
    agent any
    environment {
        STAGE = "production"
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

                    // Clone the main application repository
                    checkout([$class: 'GitSCM',
                        branches: [[name: "${GIT_BRANCH}"]],
                        userRemoteConfigs: [[url: "${GIT_URL}"]]
                    ])

                    // Clone the configuration repository and copy the correct samconfig.toml
                    sh '''
                    echo "Fetching samconfig.toml from unir-cp1-c-sam-config (${STAGE} branch)..."
                    git clone --depth 1 --branch ${STAGE} https://github.com/charlstown/unir-cp1-c-sam-config.git config_repo

                    # Move the config file to the expected location
                    mv config_repo/samconfig.toml .

                    # Clean up the temporary repo
                    rm -rf config_repo

                    echo "samconfig.toml successfully fetched for ${STAGE} environment."
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    sh '''
                    # Check AWS identity
                    aws sts get-caller-identity

                    # Build the application
                    sam build --debug

                    # Validate the CloudFormation template
                    sam validate --region ${AWS_REGION}

                    # Deploy using the downloaded samconfig.toml
                    sam deploy --region ${AWS_REGION} --no-confirm-changeset --no-fail-on-empty-changeset --debug
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

                BASE_URL=${BASE_URL} python3 -m pytest -m read --junitxml=result-rest.xml test/integration/todoApiTest.py
                '''

                // Publish REST test results
                junit 'result-rest.xml'
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
