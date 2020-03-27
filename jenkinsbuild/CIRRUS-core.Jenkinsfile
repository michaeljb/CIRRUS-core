pipeline {

  // Build on a slave with docker (for pre-req?)
  agent { label 'docker' }

  parameters {
    choice(name: 'MATURITY', choices: ['DEV', 'INT', 'TEST', 'PROD'], description: 'The MATURITY (AWS) account to deploy')
    string(name: 'DEPLOY_NAME', defaultValue: 'asf', description: 'The name of the stack for this MATURITY')
  }

  stages {
    stage('DEV Settings') {
      when { equals expected: "DEV", actual: params.MATURITY }

      params.AWS_CREDS_ID = "asf-cumulus-core-sandbox"
      params.AWS_REGION = "us-east-1"
      params.CHATHOST = "https://chat.asf.alaska.edu/hooks/dm8kzc8rxpr57xkt9w6tnfaasr"
      params.CHAT_ROOM = "RAIN"
      params.CMR_CREDS_ID = "asf-cumulus-core-cmr_creds_UAT"
      params.URS_CREDS_ID = "asf-cumulus-core-urs_creds_UAT"
      params.TOKEN_SECRET_ID = "asf-cumulus-core-token-sandbox"
      params.DAAC_REPO = "git@github.com:asfadmin/asf-cumulus-core.git"
      params.DAAC_REF = "master"
    }
    stage('INT Settings') {
      when { equals expected: "INT", actual: params.MATURITY }

      params.AWS_CREDS_ID = "asf-cumulus-core-sit"
      params.AWS_REGION = "us-east-1"
      params.CHATHOST = "https://chat.asf.alaska.edu/hooks/dm8kzc8rxpr57xkt9w6tnfaasr"
      params.CHAT_ROOM = "RAIN"
      params.CMR_CREDS_ID = "cmr-creds-uat"
      params.URS_CREDS_ID = "urs-creds-uat"
      params.TOKEN_SECRET_ID = "asf-cumulus-core-token-sit"
      params.DAAC_REPO = "git@github.com:asfadmin/asf-cumulus-core.git"
      params.DAAC_REF = "master"
    }
    stage('TEST Settings') {
      when { equals expected: "TEST", actual: params.MATURITY }
    }
    stage('PROD Settings') {
      when { equals expected: "PROD", actual: params.MATURITY }
    }

    stage('Start Cumulus Deployment') {
      steps {
        // Send chat notification
        mattermostSend channel: "${CHAT_ROOM}", color: '#EAEA5C', endpoint: "${params.CHATHOST}", message: "Build started: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>). See (<{$env.RUN_CHANGES_DISPLAY_URL}|Changes>)."
      }
    }

    stage('Clone and checkout DAAC repo/ref') {
      steps {
        sh "cd ${WORKSPACE}"
        sh "if [ ! -d \"daac-repo\" ]; then git clone ${params.DAAC_REPO} daac-repo; fi"
        sh "cd daac-repo && git fetch && git checkout ${params.DAAC_REF} && git pull && cd .."
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
        CMR_CREDS = credentials("${params.CMR_CREDS_ID}")
        URS_CREDS = credentials("${params.URS_CREDS_ID}")
        TOKEN_SECRET = credentials("${params.TOKEN_SECRET_ID}")
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${params.AWS_CREDS_ID}"]])  {

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
                                    --env AWS_REGION='${params.AWS_REGION}' \
                                    --env DAAC_REPO='${params.DAAC_REPO}' \
                                    --env DAAC_REF='${params.DAAC_REF}' \
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
      mattermostSend channel: "${CHAT_ROOM}", color: '#CEEBD3', endpoint: "${params..CHATHOST}", message: "Build Successful: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
    }
    failure {
      sh "env"
      sh "echo ${WORKSPACE}"
      sh "cd \"${WORKSPACE}\""
      sh "tree"

      mattermostSend channel: "${CHAT_ROOM}", color: '#FFBDBD', endpoint: "${params.CHATHOST}", message: "Build Failed:  🤬${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)🤬"

    }
  }

} // pipeline
