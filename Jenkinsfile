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
        
        stage('Authenticate to Conjur via REST API') {
            steps {
                script {
                    echo 'Authenticating to Conjur using REST API...'
                    withCredentials([string(credentialsId: 'conjur-api-key', variable: 'API_KEY')]) {
                        def encodedLogin = CONJUR_LOGIN.replace('/', '%2F')
                        
                        echo "Using URL: ${CONJUR_URL}/authn/${CONJUR_ACCOUNT}/${encodedLogin}/authenticate"
                        
                        // Get token and base64 encode it with newlines stripped
                        env.CONJUR_TOKEN = sh(
                            script: """
                                curl -k -d "\${API_KEY}" \
                                  "${CONJUR_URL}/authn/${CONJUR_ACCOUNT}/${encodedLogin}/authenticate" \
                                  -s | base64 | tr -d '\\r\\n'
                            """,
                            returnStdout: true
                        ).trim()
                        
                        echo 'Successfully authenticated to Conjur ✓'
                    }
                }
            }
        }
        
stage('Retrieve AWS Credentials from Conjur') {
    steps {
        script {
            echo 'Retrieving AWS credentials from Conjur...'
            
            // Get AWS Access Key ID
            env.AWS_ACCESS_KEY_ID = sh(
                script: """
                    curl -s -k \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Token token=\\"${env.CONJUR_TOKEN}\\"" \
                      "${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${AWS_ACCESS_KEY_PATH}"
                """,
                returnStdout: true
            ).trim()
            
            // DON'T echo the actual value
            echo "AWS Access Key retrieved (length: ${env.AWS_ACCESS_KEY_ID.length()})"
            
            // Get AWS Secret Access Key
            env.AWS_SECRET_ACCESS_KEY = sh(
                script: """
                    curl -s -k \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Token token=\\"${env.CONJUR_TOKEN}\\"" \
                      "${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${AWS_SECRET_KEY_PATH}"
                """,
                returnStdout: true
            ).trim()
            
            echo "AWS Secret Key retrieved (length: ${env.AWS_SECRET_ACCESS_KEY.length()})"
            
            // Get S3 Bucket Name
            env.S3_BUCKET = sh(
                script: """
                    curl -s -k \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Token token=\\"${env.CONJUR_TOKEN}\\"" \
                      "${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${BUCKET_NAME_PATH}"
                """,
                returnStdout: true
            ).trim()
            
            echo "S3 Bucket retrieved: ${env.S3_BUCKET}"
            
            // Get AWS Region
            env.AWS_REGION = sh(
                script: """
                    curl -s -k \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Token token=\\"${env.CONJUR_TOKEN}\\"" \
                      "${CONJUR_URL}/secrets/${CONJUR_ACCOUNT}/variable/${REGION_PATH}"
                """,
                returnStdout: true
            ).trim()
            
            echo "AWS Region retrieved: ${env.AWS_REGION}"
            
            echo 'Successfully retrieved all AWS credentials ✓'
        }
    }
}
        
        
        stage('Verify AWS Connection') {
            steps {
                script {
                    echo 'Testing AWS connection...'
                    sh '''
			set +x  # Disable command echoing
                        export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
                        export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
                        export AWS_DEFAULT_REGION="${AWS_REGION}"
                        
                        set +x  # Disable command echoing
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
                        set +x  # Disable command echoing
                        export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
                        export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
                        export AWS_DEFAULT_REGION="${AWS_REGION}"
                        
                        set +x  # Disable command echoing
                        # Sync website files to S3
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
                env.CONJUR_TOKEN = ''
                env.AWS_ACCESS_KEY_ID = ''
                env.AWS_SECRET_ACCESS_KEY = ''
                env.S3_BUCKET = ''
                env.AWS_REGION = ''
                echo 'Cleanup complete ✓'
            }
        }
        success {
            echo '🎉 Deployment succeeded!'
            echo 'Visit your website at the URL shown above.'
        }
        failure {
            echo '❌ Deployment failed. Check the console output for errors.'
        }
    }
}
