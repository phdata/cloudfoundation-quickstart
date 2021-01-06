pipeline {   
    environment {
     cfci_version = "1.0.0"
     project = "project"
     repo_type = "gitlab"
     deployment_decriptor = "deployment.yaml"
     plan_all = "true"
     stage = 'build'
     GITLAB_SVC_ACCNT_TOKEN = credentials('GITLAB_SVC_ACCNT_TOKEN') 
     phdata_artifactory_user = credentials('phdata_artifactory_user') 
     phdata_artifactory_secret = credentials('phdata_artifactory_secret') 
    }     
    agent any
    stages {
       stage('PRE-BUILD') {    
         when {
            expression { env.gitlabMergeRequestIid != null && (env.gitlabMergeRequestState == 'opened' || env.gitlabTriggerPhrase == 'cf deploy dev' || env.gitlabTriggerPhrase == 'cf deploy prod' ) } 
            // expression { env.gitlabMergeRequestIid != null && (env.gitlabMergeRequestState == 'opened' || env.gitlabTriggerPhrase ==~ // ) } 
          } 
          steps {
            updateGitlabCommitStatus name: 'build', state: 'pending'
            sh 'printenv'   
            sh """
            python3 -m venv env
            source env/bin/activate
            pip3 install --quiet -r requirements.txt
            """
            sh 'git clone --branch "feature/add_jenkins_support" --single-branch https://github.com/phdata/cloudfoundation-cfci-utilities.git'
            sh 'cp cloudfoundation-cfci-utilities/releases/build-tools/* .'
            sh 'cp stack_parser.py $project'
            sh 'rm -rf cloudfoundation-cfci-utilities'
            sh 'chmod +x $WORKSPACE/gitlab_jenkins_build.sh'
          }
       }
       stage('DEPLOY PLAN'){
         when {
            expression { env.gitlabMergeRequestState == 'opened' }
          } 
          environment { 
            pr_status = 'OPEN'
            }
          steps {  
            sh 'printenv'
            withCredentials([
              [$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID_prod', credentialsId: 'AWS_PROD', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY_prod'],
              [$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID_dev', credentialsId: 'AWS_DEV', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY_dev']
              ]) {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh """
                source env/bin/activate
                cd $project
                $WORKSPACE/gitlab_jenkins_build.sh
                """
              }
            }
          }
       }
       stage('Deploy to Dev'){
          when {
            expression { env.gitlabTriggerPhrase == 'cf deploy dev' &&  env.gitlabMergeRequestState != 'opened'} 
          } 
          environment {
             deploy_environment="dev"
             pr_status="MERGED"
          }
          steps {
            withCredentials([
              [$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID_prod', credentialsId: 'AWS_PROD', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY_prod'],
              [$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID_dev', credentialsId: 'AWS_DEV', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY_dev']
              ]) {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh """
                source env/bin/activate
                cd $project
                $WORKSPACE/gitlab_jenkins_build.sh
                """
              }
            }
          }
        }
      stage('Deploy to Prod'){
          when {
            // expression { env.gitlabTriggerPhrase != null && env.gitlabMergeRequestIid !=null && env.gitlabTriggerPhrase == 'cf deploy prod' &&  env.gitlabMergeRequestState != 'opened'} 
            expression { env.gitlabTriggerPhrase == 'cf deploy prod' &&  env.gitlabMergeRequestState != 'opened'} 
          } 
          environment {
             deploy_environment="prod"
             pr_status="MERGED"
          }
          steps {
            withCredentials([
              [$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID_prod', credentialsId: 'AWS_PROD', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY_prod']
              ]) {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh """
                source env/bin/activate
                cd $project
                $WORKSPACE/gitlab_jenkins_build.sh
                """
              }
            }
          }
      }
    }    
    post {
        always {  
            withEnv(['stage=post_build']) {
            // withCredentials([string(credentialsId: 'GITLAB_SVC_ACCNT_TOKEN', variable: 'GITLAB_SVC_ACCNT_TOKEN')]) 
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                  sh """
                  cd $project
                  $WORKSPACE/gitlab_jenkins_build.sh
                  """
              }
            }
            cleanWs()
        }
        success {
            updateGitlabCommitStatus name: 'build', state: 'success'
        }
        failure {
            updateGitlabCommitStatus name: 'build', state: 'failed'
        }
        // unstable {
        //     echo 'run only if the run was marked as unstable'
        // }
        // changed {
        //     echo 'run only if the state of the Pipeline has changed'
        // }
    }
}