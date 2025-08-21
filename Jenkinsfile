pipeline {
  agent any

  parameters {
    choice(name: 'ENVIRONMENT', choices: ['dev', 'preprod', 'prod'], description: 'Deployment environment')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to deploy')
  }

  environment {
    REGION = 'us-east-1'
    AWS_DEFAULT_REGION = "${REGION}"          // so aws cli doesn’t need --region every time
    ZIP_FILE = 'lambda_package.zip'
    ARN_FILE = 'lambda_arn.txt'
    SONAR_SCANNER_HOME = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
  }

  stages {

    stage('Set Environment Variables') {
      steps {
        script {
          switch (params.ENVIRONMENT) {
            case 'dev':
              env.FUNCTION_NAME   = 'hey-world-demo-dev'
              env.HANDLER         = 'test.devtest.lambda_function.lambda_handler'
              env.SCHEDULE        = 'rate(15 minutes)'
              env.ROLE_ARN        = 'arn:aws:iam::529088259986:role/service-role/s3_execRole'
              env.AWS_CRED_ID     = 'lambda-function-aws-cred-dev'
              env.EVENT_RULE_NAME = 'hello-demo-schedule-dev'
              env.SECRET_NAME     = 'my-secret-dev'
              break
            case 'preprod':
              env.FUNCTION_NAME   = 'hey-world-demo-preprod'
              env.HANDLER         = 'test.devtest.lambda_function.lambda_handler'
              env.SCHEDULE        = 'rate(10 minutes)'
              env.ROLE_ARN        = 'arn:aws:iam::529088259986:role/service-role/s3_execRole'
              env.AWS_CRED_ID     = 'lambda-function-aws-cred-preprod'
              env.EVENT_RULE_NAME = 'hello-demo-schedule-preprod'
              env.SECRET_NAME     = 'my-secret-preprod'
              break
            case 'prod':
              env.FUNCTION_NAME   = 'hey-world-demo-prod'
              env.HANDLER         = 'test.devtest.lambda_function.lambda_handler'
              env.SCHEDULE        = 'rate(5 minutes)'
              env.ROLE_ARN        = 'arn:aws:iam::529088259986:role/service-role/s3_execRole'
              env.AWS_CRED_ID     = 'lambda-function-aws-cred-prod'
              env.EVENT_RULE_NAME = 'hello-demo-schedule-prod'
              env.SECRET_NAME     = 'my-secret-prod'
              break
          }
          echo "Deploying to: ${params.ENVIRONMENT}"
          echo "Lambda: ${env.FUNCTION_NAME}"
        }
      }
    }

    stage('Clone Repository') {
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

    stage('Prepare Package') {
      steps {
        sh '''
          echo "Zipping Lambda package..."
          zip -r "$ZIP_FILE" test \
            -x "**/.git/*" \
            -x "**/README.md" \
            -x "**/Jenkinsfile"

          echo "Zip file contents:"
          unzip -l "$ZIP_FILE"
        '''
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv('sq1') {
          withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'TOKEN')]) {
            sh '''
              echo "Running SonarQube analysis..."
              "${SONAR_SCANNER_HOME}/bin/sonar-scanner" \
                -Dsonar.projectKey=aws-lambda-demo \
                -Dsonar.sources=test \
                -Dsonar.login="$TOKEN"
            '''
          }
        }
      }
    }

    stage('Deploy Lambda') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: "${env.AWS_CRED_ID}",
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
          sh '''
            echo "Checking if Lambda exists..."
            if ! aws lambda get-function --function-name "$FUNCTION_NAME" >/dev/null 2>&1; then
              echo "Creating new Lambda..."
              aws lambda create-function \
                --function-name "$FUNCTION_NAME" \
                --runtime python3.13 \
                --role "$ROLE_ARN" \
                --handler "$HANDLER" \
                --zip-file fileb://"$ZIP_FILE"
            fi

            echo "Waiting until Lambda is Active..."
            while true; do
              STATUS=$(aws lambda get-function-configuration \
                --function-name "$FUNCTION_NAME" \
                --query 'State' --output text)
              echo "Status: $STATUS"
              [ "$STATUS" = "Active" ] && break
              sleep 5
            done

            echo "Updating Lambda code..."
            aws lambda update-function-code \
              --function-name "$FUNCTION_NAME" \
              --zip-file fileb://"$ZIP_FILE"

            echo "Setting Lambda environment variables..."
            aws lambda update-function-configuration \
              --function-name "$FUNCTION_NAME" \
              --environment "Variables={SECRET_NAME=$SECRET_NAME}"

            echo "Saving Lambda ARN..."
            aws lambda get-function \
              --function-name "$FUNCTION_NAME" \
              --query 'Configuration.FunctionArn' \
              --output text > "$ARN_FILE"
          '''
        }
      }
    }

    stage('Setup CloudWatch Schedule') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: "${env.AWS_CRED_ID}",
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
          sh '''
            LAMBDA_ARN=$(cat "$ARN_FILE")

            echo "Create/Update Events rule..."
            aws events put-rule \
              --schedule-expression "$SCHEDULE" \
              --name "$EVENT_RULE_NAME"

            echo "Fetch rule ARN for scoped permission..."
            RULE_ARN=$(aws events describe-rule --name "$EVENT_RULE_NAME" --query 'Arn' --output text)

            echo "Clean old Lambda permission (if any)..."
            aws lambda remove-permission \
              --function-name "$FUNCTION_NAME" \
              --statement-id "$EVENT_RULE_NAME-stmt" || true

            echo "Add permission for Events to invoke Lambda (scoped to rule ARN)..."
            aws lambda add-permission \
              --function-name "$FUNCTION_NAME" \
              --statement-id "$EVENT_RULE_NAME-stmt" \
              --action lambda:InvokeFunction \
              --principal events.amazonaws.com \
              --source-arn "$RULE_ARN"

            echo "Attach Lambda target to rule..."
            aws events put-targets \
              --rule "$EVENT_RULE_NAME" \
              --targets "Id"="1","Arn"="$LAMBDA_ARN"
          '''
        }
      }
    }
  }

  post {
    failure {
      echo "Pipeline failed for ${params.ENVIRONMENT}"
    }
    success {
      echo "Deployment completed for ${params.ENVIRONMENT}"
    }
  }
}
