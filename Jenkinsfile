pipeline {

    // You can define a specific label for agent
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

      stage('Terraform Init'){
            steps {
                sh "terraform init -input=false -force-copy"
            }
        }

        stage('Terraform Plan'){
            steps {
                    script {
                        env.CHANGES = sh (
                            script:'terraform plan -out=tfplan -input=false -detailed-exitcode' ,
                            returnStdout: false , returnStatus: true
                        )
                        // Return Exit Code
                        // 0 = Succeeded with empty diff (no changes)
                        // 1 = Error
                        // 2 = Succeeded with non-empty diff (changes present)
                        // echo "Change Var: ${env.CHANGES}"
                        if (env.CHANGES == '1'){
                            sh "exit 1"
                        } else if (env.CHANGES == '0') {
                            echo 'NO CHANGES to apply!'
                            env.APPROVE = false
                        } else {
                            echo 'CHANGES to apply!'
                        }
                     }
            }
            post {
                success {
                        archive 'tfplan'
                }
            }
        }

        stage('Approve'){
            when {
                allOf {
                    branch 'master'
                    expression { return CHANGES ==~ /2/ }
                }
            }
            steps {
                // Slack notification when you have to approve the changes
                slackSend (channel: "#devops", color: '#4286f4', message: "Apply Changes Approval: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.JOB_DISPLAY_URL})")
                script {
                    try {
                        timeout(time:10, unit:'MINUTES') {
                            env.APPROVE_CHANGE = input message: 'Apply Changes', ok: 'Continue',
                                parameters: [choice(name: 'APPROVE_CHANGE', choices: 'YES\nNO', description: 'Apply Changes?')]
                            if (env.APPROVE_CHANGE == 'YES'){
                                env.APPROVE = true
                            } else {
                                env.APPROVE = false
                            }
                        }
                    } catch (error) {
                        env.APPROVE = false
                        echo 'Timeout has been reached!!! Apply Changes skipped'
                    }
                }
            }
        }

        stage('Terraform Apply'){
            when {
                allOf {
                    branch 'master'
                    expression { return APPROVE ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/ }
                }
            }
            steps {
                    sh "terraform apply -input=false -auto-approve tfplan"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            slackSend (channel: "#devops", color: '#d54c53', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.JOB_DISPLAY_URL})")
        }
    }

}
