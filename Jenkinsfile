// Jenkinsfile
// Pipeline: Run Ansible playbook → pull 5 lightest Docker images → push playbook + report to Git
//
// REQUIRED JENKINS CREDENTIALS (configure in Manage Jenkins → Credentials):
//   - git-credentials     : Username/Password or SSH key for your Git repo
//   - dockerhub-creds     : Username/Password for Docker Hub (optional, for private images)
//
// REQUIRED JENKINS PLUGINS:
//   - Git Plugin
//   - Credentials Binding Plugin
//   - Ansible Plugin (optional — we call ansible-playbook directly via sh)

pipeline {

    agent any   // Change to a label if you have dedicated Ansible agents

    // -----------------------------------------------------------------------
    // Configurable parameters — override at build time in Jenkins UI
    // -----------------------------------------------------------------------
    parameters {
        string(
            name: 'GIT_REPO_URL',
            defaultValue: 'https://github.com/your-org/your-repo.git',
            description: 'Git repository to push results to'
        )
        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Branch to commit and push to'
        )
        string(
            name: 'GIT_COMMIT_EMAIL',
            defaultValue: 'jenkins@yourcompany.com',
            description: 'Email used in Git commit'
        )
        string(
            name: 'GIT_COMMIT_USER',
            defaultValue: 'Jenkins CI',
            description: 'Name used in Git commit'
        )
        booleanParam(
            name: 'DOCKER_LOGIN',
            defaultValue: false,
            description: 'Enable if your images require Docker Hub authentication'
        )
    }

    environment {
        PLAYBOOK       = 'pull_lightest_images.yml'
        OUTPUT_REPORT  = 'docker_image_report.json'
        WORKSPACE_DIR  = "${env.WORKSPACE}"
    }

    // -----------------------------------------------------------------------
    // Trigger: run every day at midnight (adjust as needed)
    // -----------------------------------------------------------------------
    triggers {
        cron('H 0 * * *')
    }

    // -----------------------------------------------------------------------
    // Stages
    // -----------------------------------------------------------------------
    stages {

        stage('Checkout') {
            steps {
                echo "Checking out branch: ${params.GIT_BRANCH}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${params.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url           : params.GIT_REPO_URL,
                        credentialsId : 'git-credentials'
                    ]],
                    extensions: [[$class: 'CleanBeforeCheckout']]
                ])
            }
        }

        stage('Validate Environment') {
            steps {
                sh '''
                    echo "=== Tool versions ==="
                    ansible --version
                    docker --version
                    python3 --version
                    echo "=== Ansible collections ==="
                    ansible-galaxy collection list | grep community.docker || true
                '''
            }
        }

        stage('Install Ansible Collections') {
            steps {
                sh '''
                    ansible-galaxy collection install community.docker --upgrade
                '''
            }
        }

        stage('Docker Login') {
            when {
                expression { return params.DOCKER_LOGIN }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                sh """
                    ansible-playbook ${PLAYBOOK} \
                        -e "output_report=${WORKSPACE_DIR}/${OUTPUT_REPORT}" \
                        -v
                """
            }
            post {
                always {
                    // Archive report as a Jenkins build artifact regardless of outcome
                    archiveArtifacts artifacts: "${OUTPUT_REPORT}", allowEmptyArchive: true
                }
            }
        }

        stage('Verify Output') {
            steps {
                script {
                    def reportFile = "${WORKSPACE_DIR}/${OUTPUT_REPORT}"
                    if (!fileExists(reportFile)) {
                        error("Report file not found: ${reportFile}")
                    }
                    def report = readJSON file: reportFile
                    echo "✅ Report generated. Lightest images found: ${report.lightest_5.size()}"
                    report.lightest_5.eachWithIndex { img, idx ->
                        echo "  ${idx + 1}. ${img.name} — ${img.size_mb} MB"
                    }
                }
            }
        }

        stage('Commit & Push to Git') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'git-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh """
                        git config user.email "${params.GIT_COMMIT_EMAIL}"
                        git config user.name  "${params.GIT_COMMIT_USER}"

                        # Stage both the playbook and the generated report
                        git add ${PLAYBOOK}
                        git add ${OUTPUT_REPORT}

                        # Only commit if there are actual changes
                        if git diff --cached --quiet; then
                            echo "No changes to commit — skipping push."
                        else
                            git commit -m "ci: update lightest Docker images report [$(date -u '+%Y-%m-%d %H:%M UTC')]"

                            # Push using embedded credentials
                            REPO_URL=\$(git remote get-url origin | \
                                sed "s|https://|https://\${GIT_USER}:\${GIT_PASS}@|")
                            git push "\${REPO_URL}" HEAD:${params.GIT_BRANCH}
                            echo "✅ Changes pushed to ${params.GIT_BRANCH}"
                        fi
                    """
                }
            }
        }
    }

    // -----------------------------------------------------------------------
    // Post-pipeline notifications
    // -----------------------------------------------------------------------
    post {
        success {
            echo "✅ Pipeline completed successfully."
        }
        failure {
            echo "❌ Pipeline failed. Check logs above for details."
            // Uncomment to send email on failure:
            // mail to: 'team@yourcompany.com',
            //      subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            //      body: "Build URL: ${env.BUILD_URL}"
        }
        always {
            sh 'docker logout || true'
            cleanWs()
        }
    }
}
