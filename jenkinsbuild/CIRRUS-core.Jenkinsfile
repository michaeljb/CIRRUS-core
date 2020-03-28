pipeline {

  // Build on a slave with docker (for pre-req?)
  agent { label 'docker' }

  parameters {
    choice(name: 'MATURITY', choices: ['DEV', 'INT', 'TEST', 'PROD'], description: 'The MATURITY (AWS) account to deploy')
    string(name: 'DEPLOY_NAME', defaultValue: 'asf', description: 'The name of the stack for this MATURITY')
  }

  // Default values for DEV MATURITY
  environment {
    AWS_CREDS_ID = "asf-cumulus-core-sandbox"
    AWS_REGION = "us-east-1"
    CHATHOST = "https://chat.asf.alaska.edu/hooks/dm8kzc8rxpr57xkt9w6tnfaasr"
    CHAT_ROOM = "RAIN"
    CMR_CREDS_ID = "asf-cumulus-core-cmr_creds_UAT"
    URS_CREDS_ID = "asf-cumulus-core-urs_creds_UAT"
    TOKEN_SECRET_ID = "asf-cumulus-core-token-sandbox"
    DAAC_REPO = "git@github.com:asfadmin/asf-cumulus-core.git"
    DAAC_REF = "master"
  }

  stages {
    stage('Set Environment for MATURITY') {
      steps {
        script {
          if (params.MATURITY == 'INT') {
            AWS_CREDS_ID = "asf-cumulus-core-sit"
            AWS_REGION = "us-east-1"
            CHATHOST = "https://chat.asf.alaska.edu/hooks/dm8kzc8rxpr57xkt9w6tnfaasr"
            CHAT_ROOM = "rain"
            CMR_CREDS_ID = "asf-cumulus-core-cmr_creds_UAT"
            URS_CREDS_ID = "asf-cumulus-core-urs_creds_UAT"
            TOKEN_SECRET_ID = "asf-cumulus-core-token-sit"
            DAAC_REPO = "git@github.com:asfadmin/asf-cumulus-core.git"
            DAAC_REF = "master"
          } else if (params.MATURITY == 'TEST') {
            echo "TODO: Define ENV values for TEST"
          } else if (params.MATURITY == 'PROD') {
            echo "TODO: Define ENV values for PROD"
          }
        }
      }
    }

    stage('Start Cumulus Deployment') {
      steps {
        // Send chat notification
        mattermostSend channel: "${CHAT_ROOM}", color: '#EAEA5C', endpoint: "${CHATHOST}", message: "Build started: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>). See (<{$env.RUN_CHANGES_DISPLAY_URL}|Changes>)."
      }
    }

    stage('Clone and checkout DAAC repo/ref') {
      steps {
        sh "cd ${WORKSPACE}"
        sh "if [ ! -d \"daac-repo\" ]; then git clone ${DAAC_REPO} daac-repo; fi"
        sh "cd daac-repo && git fetch && git checkout ${DAAC_REF} && git pull && cd .."
        sh 'tree'
      }
    }
    stage('Build the CIRRUS deploy Docker image') {
      steps {
        sh """cd jenkinsbuild && \
              docker build -f nodebuild.Dockerfile -t cirrusbuilder .
           """
      }
    }
    stage('Deploy Cumulus within CIRRUS deploy Docker container') {
      environment {
        CMR_CREDS = credentials("${CMR_CREDS_ID}")
        URS_CREDS = credentials("${URS_CREDS_ID}")
        TOKEN_SECRET = credentials("${TOKEN_SECRET_ID}")
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDS_ID}"]])  {

            sh """docker run --rm   --user `id -u` \
                                    --env TF_VAR_cmr_username='${CMR_CREDS_USR}' \
                                    --env TF_VAR_cmr_password='${CMR_CREDS_PSW}' \
                                    --env TF_VAR_urs_client_id='${URS_CREDS_USR}' \
                                    --env TF_VAR_urs_client_password='${URS_CREDS_PSW}' \
                                    --env TF_VAR_token_secret='${TOKEN_SECRET}' \
                                    --env DEPLOY_NAME='${params.DEPLOY_NAME}' \
                                    --env MATURITY_IN='${params.MATURITY}' \
                                    --env AWS_ACCESS_KEY_ID='${AWS_ACCESS_KEY_ID}' \
                                    --env AWS_SECRET_ACCESS_KEY='${AWS_SECRET_ACCESS_KEY}' \
                                    --env AWS_REGION='${AWS_REGION}' \
                                    --env DAAC_REPO='${DAAC_REPO}' \
                                    --env DAAC_REF='${DAAC_REF}' \
                                    -v \"${WORKSPACE}\":/workspace \
                                    cirrusbuilder \
                                    /bin/bash /workspace/jenkinsbuild/cumulusbuilder.sh
            """

        }// withCredentials
      }// steps
    }// stage
  } // stages
  // Send build status to Mattermost, Update build badge
  post {
    always {
      sh 'echo "done"'
    }
    success {
      mattermostSend channel: "${CHAT_ROOM}", color: '#CEEBD3', endpoint: "${CHATHOST}", message: "Build Successful: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
    }
    failure {
      sh "env"
      sh "echo ${WORKSPACE}"
      sh "cd \"${WORKSPACE}\""
      sh "tree"

      mattermostSend channel: "${CHAT_ROOM}", color: '#FFBDBD', endpoint: "${CHATHOST}", message: "Build Failed:  🤬${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)🤬"

    }
  }

} // pipeline
