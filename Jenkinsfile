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
            label 'domain:core.cvent.org && os:linux'
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
            label 'domain:core.cvent.org && os:linux'
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
              image 'docker.cvent.net/cvent-cloudops/cfn-lint'
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
                              cfn_lint_errors[it] = output

                              def error_pattern = ~/^(E)\d{4}/
                              def warning_pattern = ~/^(W)\d{4}/

                              for (def message in output.split('\n\n')) {

                                def error_match = error_pattern.matcher(message)
                                def warning_match = warning_pattern.matcher(message)

                                  if (error_match.find()) {
                                      log.error 'Cfn linting', ['out': message]
                                      errored_out = true
                                  }
                                  else if (warning_match.find()) {
                                      log.warning 'Cfn linting', ['out': message]
                                  }
                                  else if (message != '') {
                                      log.info 'Cfn linting', ['out': message]
                                  }
                              }

                          }
                      }
                      catch (Exception ex) {
                          log.error "Encountered exception linting cloudformation stack ${it}: ${ex.message}"
                          if (fileExists('lint_error.log')) {
                              try {
                                  def errors_log = readFile file: 'lint_error.log'
                                  log.error 'Error Log', ['out': errors_log]
                                  sh 'rm -f lint_error.log 2> /dev/null || true'
                              }
                              catch (Exception fex) {
                                  log.error "Error reading file: ${fex.message}"
                              }
                          }
                          errored_out = true
                      }
                  }
              }
              if (cfn_lint_errors.size() > 0 && errored_out) {
                  currentBuild.result = 'FAILED'
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
                            log.error "Encountered exception yaml linting cloudformation stack ${it}: ${ex.message}"
                            if (fileExists('lint_error.log')) {
                                try {
                                    def errors_log = readFile file: 'lint_error.log'
                                    log.error 'Error Log', ['out': errors_log]
                                    sh 'rm -f lint_error.log 2> /dev/null || true'
                                }
                                catch (Exception fex) {
                                    log.error "Error reading file: ${fex.message}"
                                }
                            }
                            errored_out = true
                        }
                    }
                }
                if (yaml_lint_errors.size() > 0 && errored_out) {
                    currentBuild.result = 'FAILED'
                }
            }
          }
        }
      }
    }
    stage ('Upload templates to s3') {
      agent {
        label 'domain:core.cvent.org && os:linux'
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
            label 'master'
        }
        when { branch 'master' }
        steps {
          script {
              update_failed_stacks = []
              supported_files.findAll { file -> fileExists(file) && file.endsWith('.yaml')}.each {
                  def result = build(job: "aws-update-cloudformation-stack",
                        parameters: [
                            string(name: 'wf_requester', value: user_email.split('@')[0].toLowerCase()),
                            string(name: 'wf_ticket_number', value: ''),
                            string(name: 'wf_created_for', value: 'Cloud Automation'),
                            string(name: 'wf_business_service', value: 'Management Services'),
                            string(name: 'wf_technical_service', value: 'IAM'),
                            string(name: 'wf_icr_branch_name', value: 'master'),
                            string(name: 'wf_repository_name', value: 'iam'),
                            string(name: 'wf_branch_name', value: 'master'),
                            string(name: 'wf_build_file_name', value: ''),
                            string(name: 'wf_parameters_file_names', value: ''),
                            string(name: 'wf_nested_parameters', value: ''),
                            string(name: 'wf_max_update_query_count', value: '60'),
                            string(name: 'wf_repository_project', value: 'OPS-AWS'),
                            string(name: 'wf_cloudformation_template', value: "${it}"),
                            booleanParam(name: 'wf_notify_chat', value: false),
                            booleanParam(name: 'wf_click_to_proceed', value: false),
                            booleanParam(name: 'wf_disable_rollback', value: false)
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
            label '!master && os:linux'
        }
        when { branch 'master' }
        steps {
          script {
              def service_name = 'IAM roles, policies, or users'
              def email_to = "${user_email}"
              def email_cc = 'itautomation@cvent.com'
              def email_subject = "Encountered an issue during CI/CD automation for ${service_name}"
              if (ignored_files != null && ignored_files.size() > 0) {
                  def list_ignored = ignored_files.each {
                          return "<li>${it}</li>"
                      }.join(' ')
                  def email_message = """
                  Hi ${user_fullname.trim()},<br><br>

                  <p>Automatic update of cloudformation stacks in AWS account(s) not supported for changes made by you to ${service_name}.
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

                  <p>Automatic update of cloudformation stacks in AWS account(s) failed for changes made by you to ${service_name}.
                  Failed to update following files:</p>
                  <ul>
                  ${list_failures}
                  </ul><br>

                  <p>Changes made will have to be manually pushed. Please bear with us while one of our engineers makes time to rectify and implement the change.</p><br><br>

                  <p>Thanks,<br>
                  Cloud Automation</p>
                  """
                  email_subject = "Failed updating stacks for changes made to ${service_name}"
                  try {
                      node('!master') {
                          emailext to: email_to, mimeType: 'text/html', subject: email_subject, body: email_message, cc: email_cc
                      }
                  }
                  catch (Exception ex) {
                      log.error "Failed sending email notifications for failed updates", [ 'cause': ex.message]
                  }
                  log.error 'This build contains files/stacks that failed to update on AWS end by CI/CD', ['update_failed': update_failed_stacks]
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
          def slackMsg = "*OPS AWS - IAM Resources*\n:interrobang::interrobang::interrobang:\n\n"
          slackMsg += "<${env.BUILD_URL}|Build> to update cloudformation stacks for IAM resources failed in the following steps:\n${errors.join('\n')}"
          slackSend message: slackMsg,
                    color: 'danger',
                    channel: '#cloud-auto-internal'
        }
      }
    }
    // Commenting this section to avoid too much noise on slack channel
    // fixed {
    //   script {
    //     if (env.BRANCH_NAME == 'master') {
    //         def slackMsg = "*OPS AWS - IAM Role*\n:white_check_mark::white_check_mark::white_check_mark:\n\n"
    //         slackMsg += "<${env.BUILD_URL}|Build> to update cloudformation stacks for IAM roles succeeded"
    //         slackSend message: slackMsg,color: 'good',channel: '#cloud-auto-testing'
    //     }
    //   }
    // }
  }
}
