@Library('cloudops-base')

import cvent.opsj.core.BuildUtils;
import cvent.opsj.core.BaseUtils;

properties([
    parameters([
        string(name: 'wf_requester', defaultValue: 'OpsBotStashMan'),
        string(name: 'wf_id', defaultValue: ''),
        string(name: 'wf_output', defaultValue: '')
    ])
])

pipeline {
  agent {
    label 'domain:core.cvent.org && os:linux'
  }
  options {
    ansiColor('xterm')
    timestamps()
  }
  stages {
    stage('Build-Initiator') {
        agent {
            node {
                label 'domain:core.cvent.org && os:linux'
            }
        }
        steps {
          script {
                user_email = sh script:'echo $(git show -s --pretty=%ae)', returnStdout: true
                user_fullname = sh script:'echo $(git show -s --pretty=%an)', returnStdout: true
                // def changeAuthors = currentBuild.changeSets.collect { set ->
                //       set.collect { entry -> entry.author}//.fullName }
                //     }.flatten()
                log.info 'Build info', ['user_email': user_email, 'fullName': user_fullname]
            }
        }
    }
    stage ('Find changed files') {
        agent {
            node {
                label 'domain:core.cvent.org && os:linux'
            }
        }
        steps {
          script {
              filesChanged = BuildUtils.getFilesChanged(env.BRANCH_NAME)
              log.info "Changed files", ['files': filesChanged]
              if (filesChanged.size() == 0) {
                  error message: "No files were changed"
              }
          }
        }
    }
    stage ('Lint') {
      parallel {
        stage ('CF Lint templates') {
          agent {
            docker {
              label 'domain:core.cvent.org && os:linux'
              image 'docker.cvent.net/cvent/cfn-lint'
              alwaysPull true
              args '--entrypoint ""'
            }
          }
          steps {
            script {
              // findFiles(glob: '**/*.yaml')
              filesChanged.findAll { file -> file.endsWith('.yaml') }.each {
                sh "cfn-lint-cvent ${it}"
              }
            }
          }
        }
        stage ('YAML Lint templates') {
          agent {
            docker {
              label 'domain:core.cvent.org && os:linux'
              image 'giantswarm/yamllint'
              args '--entrypoint ""'
              alwaysPull true
            }
          }
          steps {
            script {
                rules = "{extends: relaxed, rules: {line-length: {max: 120}, new-line-at-end-of-file: disable}}"
                filesChanged.findAll { file -> file.endsWith('.yaml') }.each {
                  sh "yamllint -d \"${rules}\" ${it}"
                }
            }
          }
        }
      }
    }
    stage ('Upload templates to s3') {
      agent {
        node {
            label 'domain:core.cvent.org && os:linux'
        }
      }
      when { branch 'master' }
      steps {
        script {
            def s3Bucket = 'cvent-management-iaas-automation'
            def deploymentFolder = 'development'
            if ("${env.JENKINS_URL}".toLowerCase().indexOf('-dev.') < 1) {
                deploymentFolder = 'production'
            }
            def s3Path = "${deploymentFolder}/cloudformation/iam"
            filesChanged.findAll { file -> file.endsWith('.yaml') || file.endsWith('.json') }.each {
                awsCmd description: "Copy files to s3 bucket",
                    command: "aws s3 cp \"${it}\" \"s3://${s3Bucket}/${s3Path}/${it}\"",
                    account: 'cvent-management',
                    region: 'us-east-1'
            }
        }
      }
    }
    stage ('Publish'){
        agent {
            node {
                label 'master'
            }
        }
        when { branch 'master' }
        steps {
          script {
              update_failed_stacks = []
              filesChanged.findAll { file -> file.endsWith('.yaml')}.each {
                  def wf_requester = user_email.split('@')[0].toLowerCase()
                  wf_ticket_number = ''
                  wf_created_for = 'Cloud Automation'
                  wf_business_service = 'Management Services'
                  wf_technical_service = 'cloudops jenkins'
                  wf_icr_branch_name = 'master'
                  wf_repository_name = 'iam'
                  wf_branch_name = 'master'
                  wf_build_file_name = ''
                  wf_parameters_file_names = ''
                  wf_nested_parameters = ''
                  wf_click_to_proceed = false
                  wf_notify_chat = false
                  wf_max_update_query_count = '60'
                  wf_repository_project = 'OPS-AWS'
                  wf_disable_rollback = false
                  wf_cloudformation_template = "${it}"
                  def result = build(job: "aws-update-cloudformation-stack",
                    parameters: [
                        string(name: 'wf_requester', value: wf_requester),
                        string(name: 'wf_ticket_number', value: wf_ticket_number),
                        string(name: 'wf_created_for', value: wf_created_for),
                        string(name: 'wf_business_service', value: wf_business_service),
                        string(name: 'wf_technical_service', value: wf_technical_service),
                        string(name: 'wf_icr_branch_name', value: wf_icr_branch_name),
                        string(name: 'wf_repository_name', value: wf_repository_name),
                        string(name: 'wf_branch_name', value: wf_branch_name),
                        string(name: 'wf_build_file_name', value: wf_build_file_name),
                        string(name: 'wf_parameters_file_names', value: wf_parameters_file_names),
                        string(name: 'wf_nested_parameters', value: wf_nested_parameters),
                        string(name: 'wf_max_update_query_count', value: wf_max_update_query_count),
                        string(name: 'wf_repository_project', value: wf_repository_project),
                        string(name: 'wf_cloudformation_template', value: wf_cloudformation_template),
                        booleanParam(name: 'wf_notify_chat', value: wf_notify_chat),
                        booleanParam(name: 'wf_click_to_proceed', value: wf_click_to_proceed),
                        booleanParam(name: 'wf_disable_rollback', value: wf_disable_rollback)
                    ],
                    propagate: false,
                    wait: true)
                 def update_info = ['file': "${it}",
                        'job': result.getDisplayName(),
                        'Build_Number': result.getNumber(),
                        'BUILD_URL': result.getAbsoluteUrl(),
                        'result': result.getResult()
                    ]
                  if (result.getResult() != 'Success') {
                      update_failed_stacks += update_info
                      log.error "Update stack job failed: ", update_info
                  }
                  else {
                      log.success "Update stack job: ", update_info
                  }
              }
              if (update_failed_stacks.size() != 0) {
                  log.error "Failed updating all stacks", ['failed_updates': update_failed_stacks]
              }
          }
      }
    }
    stage ('Notification'){
        agent {
            node {
                label '!master && os:linux'
            }
        }
        when { branch 'master' }
        steps {
          script {
              // filesChanged
              // errors = BuildUtils.errorStepUrls.collect {
              //   "<${it.url}|${it.display}>"
              // }
              if (update_failed_stacks.size() != 0) {
                  service_name = 'IAM roles'
                  list_failures = update_failed_stacks.each{
                      return "<li>${it.file} : <a href=\"${it.BUILD_URL}\">${it.job}#${it.Build_Number}</a></li>"
                  }
                  email_message = """
                  This is a TEST RUN, Please Ignore
                  Hi ${user_fullname},

                  Automatic updation cloudformation stacks in AWS account(s) failed for changes made by you to IAM roles.
                  Failed to update following files:
                  <ul>
                  ${list_failures}
                  </ul>

                  Changes made will have to be manually pushed. Please bear with us while one of our engineer makes time to rectify and implement the change.

                  Thanks,
                  Cloud Automation
                  """
                  email_to = "${user_email}"
                  email_cc = "itautomation@cvent.com"
                  email_subject = "Failed updating stacks for changes made to IAM roles"
                  try {
                      node('!master') {
                          emailext to: email_to, cc: email_cc, mimeType: 'text/html', subject: email_subject, body: email_message
                      }
                  }
                  catch (Exception ex) {
                      log.error "Failed sending email notifications", [ 'cause': ex.message]
                  }
              }
          }
      }
    }
  }
  post {
    failure {
      script {
        if (env.BRANCH_NAME == 'master') {
          errors = BuildUtils.errorStepUrls.collect {
            "<${it.url}|${it.display}>"
          }

          slackSend message: "OPS AWS - IAM role update <${env.BUILD_URL}|build> failed in the following steps:\n${errors.join('\n')}",
                    color: 'danger',
                    channel: '#test-cloud-auto-iam'
        }
      }
    }
    fixed {
      script {
        if (env.BRANCH_NAME == 'master') {
          slackSend message: "Ops AWS - IAM role update  <${env.BUILD_URL}|build> succeeded",
                    color: 'good',
                    channel: '#test-cloud-auto-iam'
        }
      }
    }
  }
}
