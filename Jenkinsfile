pipeline {

    agent any

    environment {
        AWS_ECR_LOGIN = 'true'
        DOCKER_CONFIG= "${params.JENKINSHOME}"
        s3TrainUri = "${params.DATATARGETBUCKET}/${env.BUILD_ID}/input/training/"
        s3OutputKey = "${env.BUILD_ID}/input/training/"
    }

    stages {
        stage("Checkout") {
            steps {
               checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/sumirk/jenkins-mlops-sagemaker-example']]])
            }
        }

        stage("BuildPushContainer") {
            steps {
              sh """
                aws --version
                echo "${params.ECRURI}"
                aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 763104351884.dkr.ecr.ap-south-1.amazonaws.com
                
 	            docker build -t mlops-byo:${env.BUILD_ID} .
                docker tag mlops-byo:${env.BUILD_ID} ${params.ECRURI}:${env.BUILD_ID} 
                aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${params.ECRURI}

                docker push ${params.ECRURI}:${env.BUILD_ID}
                echo ${params.S3_PACKAGED_LAMBDA}
              """
            }
        }
        
        stage("ETL") {
            steps { 
              sh """
              echo "${params.DATASOURCEBUCKET}"
              echo "${params.DATATARGETBUCKET}"
              echo "${s3TrainUri}"
              aws glue start-job-run --job-name jenkins-mlops --arguments S3_INPUT_BUCKET=${params.DATASOURCEBUCKET},S3_INPUT_KEY_PREFIX="/sourcedata",S3_OUTPUT_BUCKET"=${params.DATATARGETBUCKET},S3_OUTPUT_KEY_PREFIX"="/${s3OutputKey}"             
              """
             }
        }
        
        stage("TrainModel") {
            steps { 
              sh """
               aws sagemaker create-training-job --training-job-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID} --algorithm-specification TrainingImage="${params.ECRURI}:${env.BUILD_ID}",TrainingInputMode="File" --role-arn ${params.SAGEMAKER_EXECUTION_ROLE} --input-data-config '{"ChannelName": "training", "DataSource": { "S3DataSource": { "S3DataType": "S3Prefix", "S3Uri": "${params.DATATARGETBUCKET}/"}}}' --resource-config InstanceType='ml.c4.2xlarge',InstanceCount=1,VolumeSizeInGB=5 --output-data-config S3OutputPath='${S3_MODEL_ARTIFACTS}' --stopping-condition MaxRuntimeInSeconds=3600
              """
             }
        }

      stage("TrainStatus") {
            steps {
              script {
                    def response = sh """ 
                    aws lambda invoke --function-name ${params.LAMBDA_CHECK_STATUS_TRAINING} --cli-binary-format raw-in-base64-out --region ap-south-1 --payload '{"TrainingJobName": "${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}"}' response.json
                    sleep 240
                    """
                    
                  }
              }
      }

      stage("DeployToTest") {
            steps { 
              sh """
               aws sagemaker create-model --model-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test --primary-container ContainerHostname=${env.BUILD_ID},Image=${params.ECRURI}:${env.BUILD_ID},ModelDataUrl='${S3_MODEL_ARTIFACTS}'/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}/output/model.tar.gz,Mode='SingleModel' --execution-role-arn ${params.SAGEMAKER_EXECUTION_ROLE_TEST}
               aws sagemaker create-endpoint-config --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test --production-variants VariantName='single-model',ModelName=${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test,InstanceType='ml.m4.xlarge',InitialVariantWeight=1,InitialInstanceCount=1
               aws sagemaker create-endpoint --endpoint-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test
               sleep 300
              """
             }
        }

      stage("SmokeTest") {
            steps { 
              script {
                 def response = sh """ 
                 aws lambda invoke --function-name ${params.LAMBDA_EVALUATE_MODEL} --cli-binary-format raw-in-base64-out --region ap-south-1 --payload '{"EndpointName": "'${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}'-Test", "Body": {"Payload": {"S3TestData": "${params.S3_TEST_DATA}", "S3Key": "test/iris.csv"}}}' evalresponse.json
              """
              }
            }
        }

      stage("DeployToProd") {
            steps { 
              sh """
               aws sagemaker create-model --model-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod --primary-container ContainerHostname=${env.BUILD_ID},Image=${params.ECRURI}:${env.BUILD_ID},ModelDataUrl='${S3_MODEL_ARTIFACTS}'/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}/output/model.tar.gz,Mode='SingleModel' --execution-role-arn ${params.SAGEMAKER_EXECUTION_ROLE_TEST}
               aws sagemaker create-endpoint-config --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod --production-variants VariantName='single-model',ModelName=${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod,InstanceType='ml.m4.xlarge',InitialVariantWeight=1,InitialInstanceCount=1
               aws sagemaker create-endpoint --endpoint-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod
              """
             }
        }

  }
}   
