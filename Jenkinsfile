// Jenkinsfile
// Pipeline: Run Ansible playbook → pull 5 lightest Docker images → push playbook to Git
//
// REQUIRED Jenkins credential (Manage Jenkins → Credentials):
//   - git-credentials : Username/Password for your Git repo

pipeline {

    agent any

    environment {
        PLAYBOOK   = 'pull_lightest_images.yml'
        GIT_BRANCH = 'main'
    }

    // Run every day at midnight — remove if you prefer manual triggers only
    triggers {
        cron('H 0 * * *')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Run Playbook') {
            steps {
                sh 'ansible-playbook ${PLAYBOOK}'
            }
        }

        stage('Push to Git') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'git-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh '''
                        git config user.email "jenkins@ci.local"
                        git config user.name  "Jenkins CI"

                        git add pull_lightest_images.yml

                        if git diff --cached --quiet; then
                            echo "Nothing changed, skipping push."
                        else
                            git commit -m "ci: update lightest images playbook [$(date -u +%Y-%m-%d)]"
                            REPO_URL=$(git remote get-url origin | sed "s|https://|https://${GIT_USER}:${GIT_PASS}@|")
                            git push "$REPO_URL" HEAD:main
                        fi
                    '''
                }
            }
        }
    }

    post {
        success { echo "✅ Done." }
        failure { echo "❌ Pipeline failed — check the logs." }
        always  { cleanWs() }
    }
}
