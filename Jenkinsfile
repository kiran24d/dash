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

                config = readYaml file: 'ci-config.yaml'
                jenkinsCops.whenDev {
                    log.info 'Build info', ['user_email': user_email, 'fullName': user_fullname]
                }
                ignored_files = []
                filesChanged = []
                supported_files = []
                update_failed_stacks = []
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
              // filter files as per configurations
              supported_files = []
              ignored_files = []
              filesChanged.findAll { file -> fileExists(file) && (file.endsWith('.yaml') || file.endsWith('.json')) }.each {
                  def fileparts = it.split('/')
                  if (fileparts.size() > 1) {
                      if ((config.IgnoreAccounts != null && config.IgnoreAccounts.contains(fileparts[0])) || (config.FilesToIgnore != null && config.FilesToIgnore.contains(it))) {
                          ignored_files += it
                      }
                      else {
                          supported_files += it
                      }
                  }
              }

              log.info "Filtered files", ['ci_supported_files': supported_files, 'ignoredFiles': ignored_files]
          }
        }
    }
    stage ('Lint') {
      when {
          expression { return supported_files.size() > 0 }
      }
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
              def cfn_lint_errors = [:]
              def errored_out = false
              supported_files.findAll { file -> fileExists(file) && file.endsWith('.yaml') }.each {
                  def lint_command = "cfn-lint-cvent ${it} 2>&1 > lint_error.log"
                  def fileparts = it.split('/')
                  if (fileparts.size() > 1) {
                      // account name is inside skip linting errors
                      if (config.IgnoreLintingErrors != null && config.IgnoreLintingErrors.contains(fileparts[0])) {
                          lint_command += " || true"
                      }
                      try {
                          def status = sh script: lint_command, returnStatus: true
                          if (status != 0) {
                              def output = readFile file: 'lint_error.log'
                              print(output)
                              cfn_lint_errors[it] = output
                              if (output.startsWith('error')) {
                                  log.error 'Cfn linting', ['out': output]
                                  errored_out = true
                              }
                              else if (output.contains('warning')) {
                                  log.warning 'Cfn linting', ['out': output]
                              }
                              else if (output.size() > 0) {
                                  log.info 'Cfn linting', ['out': output]
                              }
                          }
                      }
                      catch (Exception ex) {
                          log.error "Encountered exception linting cloudformation stack ${it}"//, ['errors': errors_log]
                          def errors_log = readFile file: 'lint_error.log'
                          print(errors_log)
                          sh 'rm -f lint_error.log 2> /dev/null || true'
                      }
                  }
              }
              if (cfn_lint_errors.size() > 0) {
                  if(errored_out) {
                      currentBuild.result = 'FAILED'
                  }
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
                def yaml_lint_errors = [:]
                def errored_out = false
                rules = "{extends: relaxed, rules: {line-length: {max: 120}, new-line-at-end-of-file: disable}}"
                supported_files.findAll { file -> fileExists(file) && file.endsWith('.yaml') }.each {
                    def lint_command = "yamllint -d \"${rules}\" ${it} 2>&1"
                    def fileparts = it.split('/')
                    if (fileparts.size() > 1) {
                        // account name is inside skip linting errors
                        if (config.IgnoreLintingErrors != null && config.IgnoreLintingErrors.contains(fileparts[0])) {
                            lint_command += " || true"
                        }
                        try {
                            def output = sh script: lint_command, returnStdout: true
                            print(output)
                            yaml_lint_errors[it] += output
                            if (output.contains('error')) {
                                log.error 'Yaml linting', ['out': output.split('\n')]
                                errored_out = true
                            }
                            else if (output.contains('warning')) {
                                log.warning 'Yaml linting', ['out': output.split('\n')]
                            }
                        }
                        catch (Exception ex) {
                            log.error "Encountered exception yaml linting cloudformation stack ${it}"//, ['errors': errors_log]
                            def errors_log = readFile file: 'lint_error.log'
                            print(errors_log)
                            sh 'rm -f lint_error.log 2> /dev/null || true'
                        }
                    }
                }
                if (yaml_lint_errors.size() > 0) {
                    if (errored_out) {
                        currentBuild.result = 'FAILED'
                    }
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
            jenkinsCops.whenProd {
                deploymentFolder = 'production'
            }
            def s3Path = "${deploymentFolder}/cloudformation/iam"
            supported_files.findAll { file -> fileExists(file) && (file.endsWith('.yaml') || file.endsWith('.json')) }.each {
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
              supported_files.findAll { file -> fileExists(file) && file.endsWith('.yaml')}.each {
                  wf_requester = user_email.split('@')[0].toLowerCase()
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
              def service_name = 'IAM roles'
              def email_to = "${user_email}"
              def email_cc = 'itautomation@cvent.com'
              def email_subject = "Encountered an issue during CI/CD automation for ${service_name}"
              if (ignored_files != null && ignored_files.size() > 0) {
                  def list_ignored = ignored_files.each {
                          return "<li>${it}</li>"
                      }.join(' ')
                  def email_message = """
                  Hi ${user_fullname.trim()},<br><br>

                  <p>Automatic updation of cloudformation stacks in AWS account(s) not supported for changes made by you to ${service_name}.
                  Files/stacks that were ignored by CI/CD automation are as listed below:</p>
                  <ul>
                  ${list_ignored}
                  </ul><br>

                  <p>Changes made will have to be manually pushed. Please bear with us while one of our engineers makes time to implement the change.</p><br><br>

                  <p>Thanks,<br>
                  Cloud Automation</p>
                  """
                  email_subject = "Files that were ignored by CI/CD automation for changes made to ${service_name}"
                  try {
                      node('!master') {
                          emailext to: email_to, mimeType: 'text/html', subject: email_subject, body: email_message, cc: email_cc
                      }
                  }
                  catch (Exception ex) {
                      log.error "Failed sending email notifications for ignored files", [ 'cause': ex.message]
                  }
                  log.error 'This build contains files/stacks that were ignored by CI/CD', ['ignoredFiles': ignored_files]
                  currentBuild.result = 'FAILURE'
              }
              if (update_failed_stacks != null && update_failed_stacks.size() != 0) {
                  def list_failures = update_failed_stacks.each{
                          return "<li>${it.file} : <a href=\"${it.BUILD_URL}\">${it.job}#${it.Build_Number}</a></li>"
                      }.join(' ')
                  def email_message = """
                  Hi ${user_fullname.trim()},<br><br>

                  <p>Automatic updation of cloudformation stacks in AWS account(s) failed for changes made by you to ${service_name}.
                  Failed to update following files:</p>
                  <ul>
                  ${list_failures}
                  </ul><br>

                  <p>Changes made will have to be manually pushed. Please bear with us while one of our engineers makes time to rectify and implement the change.</p><br><br>

                  <p>Thanks,<br>
                  Cloud Automation</p>
                  """
                  email_subject = "Failed updating stacks for changes made to IAM roles"
                  try {
                      node('!master') {
                          emailext to: email_to, mimeType: 'text/html', subject: email_subject, body: email_message, cc: email_cc
                      }
                  }
                  catch (Exception ex) {
                      log.error "Failed sending email notifications for failed updates", [ 'cause': ex.message]
                  }
                  log.error 'This build contains files/stacks that failed updation on AWS end by CI/CD', ['update_failed': update_failed_stacks]
                  currentBuild.result = 'FAILURE'
              }
          }
      }
    }
  }
  post {
    failure {
      script {
        if (env.BRANCH_NAME == 'master') {
          def errors = BuildUtils.errorStepUrls.collect {
            "* <${it.url}|${it.display}>"
          }
          if (ignored_files.size() > 0) {
              errors += "*Ignored Files (requires manual updates):*"
              errors += ignored_files.collect {
                  "* <https://stash/projects/OPS-AWS/repos/iam/browse/${it}?at=refs%2Fheads%2F${env.BRANCH_NAME}|${it}> \n"
              }
          }
          if (update_failed_stacks.size() > 0) {
              errors += "*Failed updates:*"
              errors += update_failed_stacks.collect {
                  "* <https://stash/projects/OPS-AWS/repos/iam/browse/${it}?at=refs%2Fheads%2F${env.BRANCH_NAME}|${it}> \n"
              }
          }
          def slackMsg = "*OPS AWS - IAM Role*\n:interrobang::interrobang::interrobang:\n\n"
          slackMsg += "<${env.BUILD_URL}|Build> to update cloudformation stacks for IAM roles failed in the following steps:\n${errors.join('\n')}"
          slackSend message: slackMsg,
                    color: 'danger',
                    channel: '#cloud-auto-testing'
        }
      }
    }
    fixed {
      script {
        if (env.BRANCH_NAME == 'master') {
            def slackMsg = "*OPS AWS - IAM Role*\n:white_check_mark::white_check_mark::white_check_mark:\n\n"
            slackMsg += "<${env.BUILD_URL}|Build> to update cloudformation stacks for IAM roles succeeded"
            slackSend message: slackMsg,color: 'good',channel: '#cloud-auto-testing'
        }
      }
    }
  }
}
