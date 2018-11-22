#!/usr/bin/env groovy

node {
    deleteDir()
    checkout scm

    wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'XTerm', 'defaultFg': 1, 'defaultBg': 2]) {
      wrap([$class: 'TimestamperBuildWrapper']) {

        def environment = docker.image('python:2-wheezy')
        def tfimg = docker.image('hashicorp/terraform:0.9.6')
        def gitUrl = sh(script: "git config remote.origin.url", returnStdout: true).trim()
        def gitCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
        def repo   = 'github.com/samindr/MyGit'

          // run as user 0 group 0 - see comment in Setup Virtualenv stage
          environment.inside('-u 0:0') {
            withEnv(["GIT_COMMIT=${gitCommit}"]) {
              stage('Setup Virtualenv') {
                /*
                 * If we run the container as 1000:1000 (which Jenkins will do by
                 * default), we can't pip install from a git URL in the container,
                 * because we're running as a user not present in /etc/passwd. But
                 * running as 0:0 has the side effect that any files we create
                 * in `pwd` (the workspace, mounted RW into the container) will be
                 * owned 0:0 (even on the host)... which means deleteDir() will fail
                 * with permissions errors. The simple solution is to not touch
                 * anything in the workspace at all; copy it to a path that exists
                 * only in the container and mess with it there.
                 */
                sh "mkdir /app && cp -a . /app && cd /app && virtualenv --no-site-packages -p python2.7 ."
              }
              stage('Install tox') {
                sh "cd /app && . bin/activate && pip install tox"
              }
              stage('Test policygen.py') {
                sh "cd /app && . bin/activate && tox"
              }
              stage('Test sqs_splunk_lambda') {
                sh "cd /app/sqs_splunk_lambda && . ../bin/activate && tox"
              }
              stage('Validate') {
                sh "cd /app && make validate && cat custodian.yml && cat policies.rst"
                sh "cp /app/custodian.yml . && cp /app/policies.rst ."
                archiveArtifacts artifacts: "custodian.yml,policies.rst", allowEmptyArchive: true
              }

              stage('Lambda Garbage Collection Dry Run') {
                sh "cd /app && make mugc-dryrun"
              }

              stage('Dry Run') {
                sh "cd /app && make dryrun"
                sh "cp -a /app/dryrun . && chown -R 1000:1000 ."
                archiveArtifacts artifacts: "custodian.yml,policies.rst,dryrun/**/*", allowEmptyArchive: true
              }

              if (env.BRANCH_NAME == 'master') {
                stage('Lambda Garbage Collection') {
                  sh "cd /app && make mugc"
                }
                stage('Install and Provision Mailer') {
                  sh "cd /app && make run mailer"
                }
              } else {
                stage('Install Mailer') {
                  // not master - install deps but don't run mailer
                  sh "cd /app && make mailerdeps && cat mailer.yml"
                }
              }

              // sqs_splunk_lambda needs Vault creds
              def secrets =  [
                  [
                    $class: 'VaultSecret',
                    path: 'some/path',
                    secretValues: [
                      // redacted; get some secrets, and set them in the environment
                    ]
                  ]
              ]

              wrap([$class: 'VaultBuildWrapper', vaultSecrets: secrets]) {
                if (env.BRANCH_NAME == 'master') {
                  stage('Install and provision sqs_splunk_lambda') {
                    sh "cd /app/sqs_splunk_lambda && . ../bin/activate && python setup.py develop"
                    sh "cd /app && . bin/activate && sqs_splunk_lambda -c sqs_splunk_lambda.yml --update-lambda"
                  }
                } else {
                  stage('Install and verify sqs_splunk_lambda') {
                    sh "cd /app/sqs_splunk_lambda && . ../bin/activate && python setup.py develop"
                    sh "cd /app && . bin/activate && sqs_splunk_lambda -c sqs_splunk_lambda.yml --validate"
                  }
                }
              } // wrap

              stage('Build Docs') {
                sh "cd /app && make docs"
              }

              if (env.BRANCH_NAME == 'master') {
                stage('Publish Docs') {
                  withCredentials([usernamePassword(credentialsId: 'SomeID', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                    sh("""
                      cd /app/docs/_build/
                      git init
                      git config user.name "jenkins"
                      git config user.email "jenkins@example.com"
                      git remote add origin git@${repo}
                      git checkout -b gh-pages

                      git add --all
                      git commit -m "docs published by ${env.BUILD_URL}"
                      git push -f "https://${env.GIT_PASS}@${repo}" HEAD:gh-pages
                    """)
                  } // withCredentials
                } // stage('Publish Docs')

                stage('Apply Job DSL for errorscan.py cron-based Jenkins job') {
                  build job: 'SeedJob', parameters: [
                    string(name: 'APP_NAME', value: 'cloud-custodian'),
                    string(name: 'APP_REPO', value: 'git@github.com:example/custodian-config'),
                    string(name: 'PATH_TO_DSL', value: 'config/pipeline.groovy'),
                    string(name: 'APP_REPO_BRANCH', value: 'master')
                  ]
                } // stage
              } // if env.BRANCH_NAME == 'master'
            } // withEnv
          } //environment
          currentBuild.result = 'SUCCESS'
        } catch(Exception ex) {
          echo "Caught exception: ${ex.toString()}"
          currentBuild.result = 'FAILURE'
          throw ex
        } finally {
          echo "Build result: ${currentBuild.result}"
          if (currentBuild.result == 'SUCCESS') {
            slackSend(color: 'good', message: "SUCCESS: ${env.JOB_NAME} <${env.BUILD_URL}|build ${env.BUILD_NUMBER}>")
          } else {
            slackSend(color: 'danger', message: "FAILED: ${env.JOB_NAME} <${env.BUILD_URL}|build ${env.BUILD_NUMBER}>")
          }
        }

      } // wrap TimestamperBuildWrapper
    } // wrap AnsiColorBuildWrapper
}
