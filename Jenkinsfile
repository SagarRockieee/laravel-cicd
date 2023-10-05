//Pipe Line
pipeline {
    agent any
    stages {
        //Stage to verify that docker is installed
        stage("Verify tooling") {
            steps {
                sh '''
                    docker info
                    docker version
                    docker compose version
                '''
            }
        }
        //Server connnection
        stage("Verify SSH connection to server") {
            steps {
                sshagent(credentials: ['aws-ec2']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@13.40.116.143 whoami
                    '''
                }
            }
        }
        //Stage to clear all the present docker containers
        stage("Clear all running docker containers") {
            steps {
                script {
                    try {
                        sh 'docker rm -f $(docker ps -a -q)'
                    } catch (Exception e) {
                        echo 'No running container to clear up...'
                    }
                }
            }
        }
        //Stage to start docker
        stage("Start Docker") {
            steps {
                sh 'make up'
                sh 'docker compose ps'
            }
        }
        //Stage for build(PHP Larvel)
        stage("Run Composer Install") {
            steps {
                sh 'docker compose run --rm composer install'
            }
        }
        //Stage to get .env file
        stage("Populate .env file") {
            steps {
                dir("/var/lib/jenkins/workspace/envs/laravel-test") {
                    fileOperations([fileCopyOperation(excludes: '', flattenFiles: true, includes: '.env', targetLocation: "${WORKSPACE}")])
                }
            }
        }
        //Automated Tests to setup in the pipeline
        stage("Run Tests") {
            steps {
                sh 'docker compose run --rm artisan test'
            }
        }
    }
    post {
        success {
            sh 'cd "/var/lib/jenkins/workspace/LaravelTest"'
            sh 'rm -rf artifact.zip'
            sh 'zip -r artifact.zip . -x "*node_modules**"'
            withCredentials([sshUserPrivateKey(credentialsId: "aws-ec2", keyFileVariable: 'keyfile')]) {
                sh 'scp -v -o StrictHostKeyChecking=no -i ${keyfile} /var/lib/jenkins/workspace/LaravelTest/artifact.zip ec2-user@13.40.116.143:/home/ec2-user/artifact'
            }
            sshagent(credentials: ['aws-ec2']) {
                sh 'ssh -o StrictHostKeyChecking=no ec2-user@13.40.116.143 unzip -o /home/ec2-user/artifact/artifact.zip -d /var/www/html'
                script {
                    try {
                        sh 'ssh -o StrictHostKeyChecking=no ec2-user@13.40.116.143 sudo chmod 777 /var/www/html/storage -R'
                    } catch (Exception e) {
                        echo 'Some file permissions could not be updated.'
                    }
                }
            }
        // Send Teams notification on successful build
        script {
            def teamsUrl = 'YOUR_TEAMS_INCOMING_WEBHOOK_URL'
            def payload = [
                '@type': 'MessageCard',
                '@context': 'http://schema.org/extensions',
                'themeColor': '0076D7',
                'summary': 'Hi Team, the Build is Successful',
                'sections': [
                    [
                        'activityTitle': "Jenkins Build Successful",
                        'facts': [
                            ['name': 'Job', 'value': env.JOB_NAME],
                            ['name': 'Build Number', 'value': env.BUILD_NUMBER],
                            ['name': 'Commit', 'value': env.GIT_COMMIT],
                            ['name': 'Built By', 'value': env.BUILD_USER_ID],
                            ['name': 'Result', 'value': env.RESULT],
                            ['name': 'Build URL', 'value': env.BUILD_URL]
                        ]
                    ]
                ]
            ]
            httpRequest(
                contentType: 'APPLICATION_JSON',
                httpMode: 'POST',
                requestBody: JsonOutput.toJson(payload),
                url: teamsUrl
            )
        }
    }
        always {
            sh 'docker compose down --remove-orphans -v'
            sh 'docker compose ps'
        }
    }
}