pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter Git branch to build', trim: true)
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose deployment environment')
    }

    environment {
        FUNCTION_NAME = "hello-world"
        REGION = "us-east-1"

        // Map Jenkins credentials per environment
        AWS_CRED_DEV   = "aws-cred-dev"
        AWS_CRED_STAGE = "aws-cred-stage"
        AWS_CRED_PROD  = "aws-cred-prod"
        SONAR_TOKEN    = credentials('SONAR_TOKEN')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: "*/${params.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/AngelsTech/aws-lambda-demo.git',
                        credentialsId: '66142c87-d271-41f4-82d1-cbbee8e844d0'
                    ]]
                ])
            }
        }

        // stage('SonarQube Analysis') {
        //     environment {
        //         SCANNER_HOME = tool 'sonar-scanner'
        //     }
        //     steps {
        //         withSonarQubeEnv('SonarQube') {
        //             sh '''
        //             ${SCANNER_HOME}/bin/sonar-scanner \
        //               -Dsonar.projectKey=aws-lambda-demo \
        //               -Dsonar.sources=. \
        //               -Dsonar.host.url=http://your-sonar-server:9000 \
        //               -Dsonar.login=${SONAR_TOKEN}
        //             '''
        //         }
        //     }
        // }

        stage('Deploy Lambda') {
            steps {
                script {
                    def awsCred = ''
                    if (params.ENV == 'dev') {
                        awsCred = env.AWS_CRED_DEV
                    } else if (params.ENV == 'stage') {
                        awsCred = env.AWS_CRED_STAGE
                    } else if (params.ENV == 'prod') {
                        awsCred = env.AWS_CRED_PROD
                    }

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh """
                        zip -j function.zip lambda_function.py
                        aws lambda update-function-code --function-name ${FUNCTION_NAME}-${params.ENV} --zip-file fileb://function.zip
                        """
                    }
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                script {
                    def awsCred = ''
                    if (params.ENV == 'dev') {
                        awsCred = env.AWS_CRED_DEV
                    } else if (params.ENV == 'stage') {
                        awsCred = env.AWS_CRED_STAGE
                    } else if (params.ENV == 'prod') {
                        awsCred = env.AWS_CRED_PROD
                    }

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh """
                        aws events put-rule --name "trigger-${FUNCTION_NAME}-${params.ENV}" --schedule-expression "rate(5 minutes)" || true
                        aws lambda add-permission --function-name ${FUNCTION_NAME}-${params.ENV} \
                          --statement-id "event-${params.ENV}" \
                          --action 'lambda:InvokeFunction' \
                          --principal events.amazonaws.com \
                          --source-arn arn:aws:events:${env.REGION}:$(aws sts get-caller-identity --query Account --output text):rule/trigger-${FUNCTION_NAME}-${params.ENV} || true

                        aws events put-targets --rule "trigger-${FUNCTION_NAME}-${params.ENV}" \
                          --targets "Id"="1","Arn"="$(aws lambda get-function --function-name ${FUNCTION_NAME}-${params.ENV} --query 'Configuration.FunctionArn' --output text)" || true
                        """
                    }
                }
            }
        }
    }
}
