pipeline {
    agent any
    
    environment {
        CONJUR_URL = 'https://conjursecrets:8443'
        CONJUR_ACCOUNT = 'myConjurAccount'
        CONJUR_LOGIN = 'host/jenkins-hosts/debian-jenkins'
        
        AWS_ACCESS_KEY_PATH = 'jenkins-app/aws/access-key-id'
        AWS_SECRET_KEY_PATH = 'jenkins-app/aws/secret-access-key'
        BUCKET_NAME_PATH = 'jenkins-app/aws/bucket-name'
        REGION_PATH = 'jenkins-app/aws/region'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out code from Git...'
                checkout scm
            }
        }
        
        stage('Retrieve AWS Credentials from Conjur via REST') {
            steps {
                script {
                    echo 'Retrieving AWS credentials from Conjur...'
                    
                    withCredentials([string(credentialsId: 'conjur-api-key', variable: 'API_KEY')]) {
                        // Use curl with Basic Auth (API key as password)
                        def login = env.CONJUR_LOGIN
                        
                        env.AWS_ACCESS_KEY_ID = sh(
                            script: """
                                curl -k -s -u '${login}:\${API_KEY}' \
                                  '${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${AWS_ACCESS_KEY_PATH}'
                            """,
                            returnStdout: true
                        ).trim()
                        
                        env.AWS_SECRET_ACCESS_KEY = sh(
                            script: """
                                curl -k -s -u '${login}:\${API_KEY}' \
                                  '${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${AWS_SECRET_KEY_PATH}'
                            """,
                            returnStdout: true
                        ).trim()
                        
                        env.S3_BUCKET = sh(
                            script: """
                                curl -k -s -u '${login}:\${API_KEY}' \
                                  '${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${BUCKET_NAME_PATH}'
                            """,
                            returnStdout: true
                        ).trim()
                        
                        env.AWS_REGION = sh(
                            script: """
                                curl -k -s -u '${login}:\${API_KEY}' \
                                  '${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${REGION_PATH}'
                            """,
                            returnStdout: true
                        ).trim()
                        
                        echo 'Successfully retrieved all AWS credentials ✓'
                    }
                }
            }
        }
        
        stage('Verify AWS Connection') {
            steps {
                script {
                    echo 'Testing AWS connection...'
                    sh '''
                        export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
                        export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
                        export AWS_DEFAULT_REGION="${AWS_REGION}"
                        
                        aws sts get-caller-identity
                        aws s3 ls s3://${S3_BUCKET}
                    '''
                    echo 'AWS connection verified ✓'
                }
            }
        }
        
        stage('Deploy to S3') {
            steps {
                script {
                    echo 'Deploying website to S3...'
                    sh '''
                        export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
                        export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
                        export AWS_DEFAULT_REGION="${AWS_REGION}"
                        
                        aws s3 sync . s3://${S3_BUCKET}/ \
                            --exclude ".git/*" \
                            --exclude "Jenkinsfile" \
                            --exclude "README.md" \
                            --delete \
                            --cache-control "max-age=3600"
                        
                        echo "Deployment complete!"
                        echo "Website URL: http://${S3_BUCKET}.s3-website-${AWS_REGION}.amazonaws.com"
                    '''
                    echo 'Successfully deployed to S3 ✓'
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo 'Cleaning up sensitive data...'
                env.AWS_ACCESS_KEY_ID = ''
                env.AWS_SECRET_ACCESS_KEY = ''
                env.S3_BUCKET = ''
                env.AWS_REGION = ''
                echo 'Cleanup complete ✓'
            }
        }
        success {
            echo '🎉 Deployment succeeded!'
        }
        failure {
            echo '❌ Deployment failed.'
        }
    }
}
