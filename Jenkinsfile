pipeline {
    agent any

    environment {
        PLAYBOOK   = 'playbook.yml'
        GIT_BRANCH = 'main'
    }

    // Run every day at midnight — remove if you prefer manual triggers only
    triggers {
        cron('H 0 * * *')  // Optional: automatically run on a schedule
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm  // Pull the latest code from the GitHub repo
            }
        }

        stage('Run Playbook') {
            steps {
                sh 'ansible-playbook ${PLAYBOOK}'  // Execute your playbook
            }
        }
    }

    post {
        success { 
            echo "✅ Playbook executed successfully." 
        }
        failure { 
            echo "❌ Pipeline failed — check the logs." 
        }
        always {
            cleanWs()  // Clean workspace after the build
        }
    }
}

