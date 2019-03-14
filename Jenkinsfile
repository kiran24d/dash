@Library('pipeline-utils') _

import cvent.jenkins.BuildUtils;

pipeline {
  agent none
  options {
    ansiColor('xterm')
    timestamps()
  }
  stages {
    stage ('Lint') {
      parallel {
        stage ('CF Lint templates') {
          agent {
            docker {
              image 'docker.cvent.net/cvent/cfn-lint'
              alwaysPull true
              args '--entrypoint ""'
            }
          }
          steps {
            script {
              findFiles(glob: '**/*.yaml').each {
                sh "cfn-lint-cvent ${it.path} || true"
              }
            }
          }
        }
        stage ('YAML Lint templates') {
          agent {
            docker {
              image 'giantswarm/yamllint'
              args '--entrypoint ""'
              alwaysPull true
            }
          }
          steps {
            sh '''yamllint $(find . -name '*.yaml') || true'''
          }
        }
      }
    }
    stage ('Publish') {
      agent {
        docker {
          image 'cvent/aws-cli'
          args '--entrypoint ""'
        }
      }
      when { branch 'master' }
      steps {
        withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'cvent-management-shared-jenkins',
        ]]) {
          script {
            findFiles(glob: '*/*/*/*.yaml').each {
              def (account, region, vpc, filename) = it.path.tokenize('/')

              shortRegion = region.replace("-", "")

              s3Bucket = 'cvent-management-iaas-automation'
              s3Path = "production/cloudformation/iam"

              s3CopyCmd = "aws s3 cp ${it.path} s3://${s3Bucket}/${s3Path}"
              echo s3CopyCmd
              // sh s3CopyCmd
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

          slackSend message: "@help-icr Ops AWS <${env.BUILD_URL}|build> failed in the following steps:\n${errors.join('\n')}",
                    color: 'danger',
                    channel: '#icr'
        }
      }
    }
    fixed {
      script {
        if (env.BRANCH_NAME == 'master') {
          slackSend message: "Ops AWS <${env.BUILD_URL}|build> succeeded",
                    color: 'good',
                    channel: '#icr'
        }
      }
    }
  }
}