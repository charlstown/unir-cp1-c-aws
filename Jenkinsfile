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
                        userRemoteConfigs: [[url: 'https://github.com/charlstown/unir-cp01-a.git']]
                    ])
                }
            }
        }
    }
}
