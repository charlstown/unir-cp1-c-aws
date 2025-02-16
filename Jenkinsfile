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

                    // Clone the repository and check out the specified branch
                    checkout([$class: 'GitSCM',
                        branches: [[name: "${GIT_BRANCH}"]],
                        userRemoteConfigs: [[url: "${GIT_URL}"]]
                    ])
                }
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