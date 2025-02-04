pipeline {
    agent any
    stages {
        stage('Get Code') {
            steps {
                script {
                    // Add the workspace as a safe directory for Git
                    sh 'git config --global --add safe.directory ${WORKSPACE}'

                    // Clone the repository and check out the specified branch
                    checkout([$class: 'GitSCM',
                        branches: [[name: "${GIT_BRANCH}"]],
                        userRemoteConfigs: [[url: 'https://github.com/charlstown/unir-cp01-a.git']]
                    ])
                }
            }
        }
        stage('Unit') {
            environment {
                PYTHONPATH="${WORKSPACE}"
            }
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    // Run Pytest unit tests with coverage
                    sh 'python3 -m coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest --junitxml=result-unit.xml test/unit'
                }
                // Publish Unit test results
                junit 'result-unit.xml'
            }
        }
        stage('Rest') {
            environment {
                PYTHONPATH="${WORKSPACE}"
            }
            steps {
                // Continue the pipeline even if REST tests fail
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    // Start Flask server in the background
                    sh '''
                    export FLASK_APP=app/api.py
                    echo "Starting Flask server..."
                    flask run --host=127.0.0.1 --port=5000 > flask.log 2>&1 &
                    echo $! > flask.pid
                    '''

                    // Readiness check for Flask server
                    sh '''
                    echo "Waiting for Flask to be ready..."
                    for i in {1..10}; do
                        if curl -s http://127.0.0.1:5000 > /dev/null; then
                            echo "Flask is up and running!"
                            break
                        fi
                        echo "Flask not ready yet, retrying in 3 seconds..."
                        sleep 3
                    done
                    '''

                    // Run REST tests
                    sh '''
                    python3 -m pytest --junitxml=result-rest.xml test/rest
                    '''
                }
                // Publish REST test results, even if some tests failed
                junit 'result-rest.xml'
            }
        }
        stage('Static') {
            steps {
                // Run flake8 with pylint
                sh 'python3 -m flake8 --format=pylint --exit-zero --output-file=result-flake8.out app'

                // Publish the flake8 report
                recordIssues tools: [flake8(pattern: 'result-flake8.out')], 
                             qualityGates: [
                                 [threshold: 8, type: 'TOTAL', unstable: true],
                                 [threshold: 10, type: 'TOTAL', unhealthy: true]
                             ]
            }
        }
        stage('Security') {
            steps {
                // Run bandit and ignore its exit code
                sh '''
                bandit -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}" || true
                '''

                // Publish the Bandit report and evaluate the quality gate
                recordIssues tools: [pyLint(name: 'bandit', pattern: 'bandit.out')], 
                            qualityGates: [
                                [threshold: 2, type: 'TOTAL', unstable: true],  // Unstable if >= 2 issues
                                [threshold: 4, type: 'TOTAL', unhealthy: true] // Fail if >= 4 issues
                            ]
            }
        }
        stage('Coverage') {
            steps {
                // Generate the coverage XML report
                sh 'python3 -m coverage xml -o coverage.xml'
                
                // Publish the coverage report
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    cobertura coberturaReportFile: 'coverage.xml', 
                              lineCoverageTargets: '95,85,85', 
                              conditionalCoverageTargets: '90, 80, 80',
                              onlyStable: false
                }
            }
        }
        stage('Performance') {
            steps {
                // Generate the coverage XML report
                sh 'jmeter -n -t test/jmeter/flask.jmx -f -l flask.jtl'

                // Publish the performance report
                perfReport sourceDataFiles: 'flask.jtl'

                // Stop Flask server after tests
                sh '''
                if [ -f flask.pid ]; then
                    echo "Stopping Flask server..."
                    kill $(cat flask.pid)
                    rm flask.pid
                fi
                '''
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
