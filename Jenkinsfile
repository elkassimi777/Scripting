<<<<<<< HEAD
pipeline{

  agent any
  stages {
      stage ("build"){
		
	   steps{
		echo 'building the application'
	    }
	}
	
	stage ("test") {
		
	     steps {
		echo 'testing the app'
		}
	}
	
	stage ("deploy"){
		
	      steps {
		echo 'testing the app'
		}
	}

  }
=======
// Uses Declarative syntax to run commands inside a container.
@Library('main-library@sl-refactoring-v2') _

pipeline {
    agent {
        kubernetes {
            yaml podCustomTemplate(builder_version="7.4-pc412-0.1.6-builder", subversion="", env=env)
            defaultContainer 'app-sigman-tester'
        }
    }
    options {
        disableConcurrentBuilds()
    }
    stages {
        stage('Checkout') {
            steps {
                println 'GIT_BRANCH = ' + env.BRANCH_NAME
            }
        }

        stage('Build') {
            steps {
                script {
                    env.GITREF = sh(script: 'git show-ref',
                        returnStdout: true
                    ).trim()
                    buildFor = 'test'
                    coverage = false
                    if (env.BRANCH_NAME ==~ /^(develop)$/){
                      coverage = true
                    }
                }
                println 'GITREF is:' + env.GITREF
                container('app-sigman-tester') {
                    sh '''
                if [ -d '/cache/$BRANCH_NAME/phpunit' ]; then
                    mkdir var
                    cp -r /cache/$BRANCH_NAME/phpunit serv/var/phpunit
                else
                    exit 0
                fi
                '''
                    buildTest(project='app-sigman', buildFor=buildFor, this)
                }
            }
        }

        stage('Phpstan Serv') {
          when { expression { env.BRANCH_NAME ==~ /^(master|develop|ci\/INF-61|PR-[0-9]+)$/ } }
          steps {
            sh 'serv/vendor/bin/phpstan analyse --error-format=junit -c phpstan-serv.neon --memory-limit 3G > phpstan-serv.xml || true'
            withChecks('Phpstan Serv'){
              junit testResults: 'phpstan-serv.xml', allowEmptyResults: true
            }
          }
        }

        stage('Phpstan Serv SRC-MODELS') {
          when { expression { env.BRANCH_NAME ==~ /^(master|develop|ci\/INF-61|PR-[0-9]+)$/ } }
          steps {
            sh 'serv/vendor/bin/phpstan analyse --error-format=junit -c phpstan-serv-src-models.neon --memory-limit 3G > phpstan-serv-src-models.xml || true'
            withChecks('Phpstan Serv'){
              junit testResults: 'phpstan-serv-src-models.xml', allowEmptyResults: true
            }
          }
        }

        stage('Phpstan Manage') {
          when { expression { env.BRANCH_NAME ==~ /^(master|develop|ci\/INF-61|PR-[0-9]+)$/ } }
          steps {
            sh 'serv/vendor/bin/phpstan --version'
            sh 'serv/vendor/bin/phpstan analyse --error-format=junit -c phpstan-manage.neon --memory-limit 3G > phpstan-manage.xml || true'
            withChecks('Phpstan Manage'){
              junit testResults: 'phpstan-manage.xml', allowEmptyResults: true
            }
          }
        }

        stage('Phpstan Sign') {
          when { expression { env.BRANCH_NAME ==~ /^(master|develop|ci\/INF-61|PR-[0-9]+)$/ } }
          steps {
            sh 'serv/vendor/bin/phpstan analyse --error-format=junit -c phpstan-sign.neon --memory-limit 3G > phpstan-sign.xml || true'
            withChecks('Phpstan Sign'){
              junit testResults: 'phpstan-sign.xml', allowEmptyResults: true
            }
          }
        }

        stage('php-cs-fixer') {
          when { expression { env.BRANCH_NAME ==~ /^(master|develop|ci\/INF-61|PR-[0-9]+)$/ } }
          steps {
            sh "serv/vendor/bin/php-cs-fixer fix --verbose --cache-file=/cache/${env.BRANCH_NAME}/php-cs.cache --dry-run --format=junit > php-cs-fixer-results.xml || true"
            withChecks('Php-CS-Fixer'){
              junit testResults: 'php-cs-fixer-results.xml', allowEmptyResults: true
            }
          }
        }

        stage('Tests') {
            when { expression { env.BRANCH_NAME != 'phalcon5' } }
            parallel {
                stage('Sign') {
                    steps {
                      container('app-sigman-tester') {
                        sh "serv/vendor/bin/phpunit --configuration phpunit-sign.xml.dist ${!coverage ? '--no-coverage' : '--coverage-clover coverage-serv.xml'} --cache-result --log-junit report-sign-unit.xml --testsuite=Unit${ignoreErrorResult(env.BRANCH_NAME)}"
                      }
                      withChecks('Unit Tests Sign'){
                        junit testResults: 'report-sign-unit.xml', allowEmptyResults: true
                      }
                    }
                }
                stage('Manage') {
                    steps {
                      container('app-sigman-tester') {
                        sh "serv/vendor/bin/phpunit --configuration phpunit-manage.xml.dist ${!coverage ? '--no-coverage' : '--coverage-clover coverage-manage.xml'} --cache-result --log-junit report-manage-unit.xml --testsuite=Unit${ignoreErrorResult(env.BRANCH_NAME)}"
                      }
                      withChecks('Unit Tests Manage'){
                        junit testResults: 'report-manage-unit.xml', allowEmptyResults: true
                      }
                    }
                }
                stage('Serv') {
                    steps {
                      container('app-sigman-tester') {
                        sh "serv/vendor/bin/phpunit --configuration phpunit-serv.xml.dist ${!coverage ? '--no-coverage' : '--coverage-clover coverage-sign.xml'} --cache-result --log-junit report-serv-unit.xml --testsuite=Unit${ignoreErrorResult(env.BRANCH_NAME)}"
                      }
                      withChecks('Unit Tests Serv'){
                        junit testResults: 'report-serv-unit.xml', allowEmptyResults: true
                      }
                    }
                }
            }
        }

        stage('Post Tests') {
            when { expression { env.BRANCH_NAME != 'phalcon5' } }
            steps {
              container('app-sigman-tester') {
                sh '''if [ -d '/cache/$BRANCH_NAME/phpunit' ]; then
                        rm -rf /cache/$BRANCH_NAME/phpunit
                      else
                        mkdir -p /cache/$BRANCH_NAME/phpunit
                      fi
                '''
                sh 'cp -r serv/var/phpunit /cache/$BRANCH_NAME/phpunit'
              }
            }
        }

        stage('SonarQube') {
          when { expression { env.BRANCH_NAME ==~ /^(develop)$/ } }
          steps {
            script{
              def scannerHome = tool 'SonarScanner';
              withSonarQubeEnv('SonarQube-Catchim') {
                sh """
                  export LANG='fr_FR'
                  export LANGUAGE='fr_FR'
                  export LC_ALL='UTF-8'
                  export LC_NAME='UTF-8'
                  export LC_CTYPE='UTF-8'
                  export SONAR_SCANNER_HOME=/cache/sonar/
                  ${scannerHome}/bin/sonar-scanner -Dproject.settings=./sonar.properties
                """
              }
            }
          }
        }
        
        stage('Check Old Angular'){
          when { expression { env.BRANCH_NAME ==~ /^(alpha|hotfix\/.*|env\/SANDBOX|env\/RECETTE|release|V(([0-9]+)((\.[0-9]+)){1,3}))$/ } }
          steps{
            script {
                env.ENV = utils(utility='getEnvFromRef', param1='app-sigman', param2=env.GITREF, this)
                env.ENVIRONMENT = env.ENV
                env.APP_NAME = "SIGMAN"
                env.AWS_ARTIFACT_BUCKET='peopulse-deploy-artifacts'
                try {
                    utils(utility='checkArtifacts', param1='na', param2='na', this)
                } catch (err) {
                    println err
                    build(job: 'Legacy NG/' + java.net.URLEncoder.encode(env.BRANCH_NAME, 'UTF-8'), wait: true)
                }
            }
          }
        }

        stage ('Angular Artifacts'){
          when { expression { env.BRANCH_NAME ==~ /^(alpha|hotfix\/.*)$/ } }
          steps {
            container('app-sigman-tester') {
              utils(utility='getFrontArtifacts', param1='app-sigman', param2='na', this)
            }
          }
        }

        stage('Build Angular') {
          when { expression { env.BRANCH_NAME ==~ /^(alpha|hotfix\/.*|env\/SANDBOX|env\/RECETTE|release|V(([0-9]+)((\.[0-9]+)){1,3}))$/ } }
          parallel {
              stage ('Sign'){
                  steps {
                    container('app-sigman-tester') {
                      buildApp(project='app-sigman-sign-angular', buildFor='Legacy Angular', this)
                    }
                  }
              }
              stage ('Manage'){
                  steps {
                    container('app-sigman-tester') {
                      buildApp(project='app-sigman-manage-angular', buildFor='Legacy Angular', this)
                    }
                  }
              }
          }
        }

        stage('Deploy') {
            when { expression { env.BRANCH_NAME ==~ /^(alpha|phalcon5|hotfix\/.*|env\/SANDBOX|env\/RECETTE|release|V(([0-9]+)((\.[0-9]+)){1,3}))$/ } }
            steps {
                container('app-sigman-tester') {
                    buildApp(project='app-sigman', buildFor='deploy', this)
                    deployApp(env, this)
                }
            }
        }
    }
    post {
        always {
            teamsNotify('app-sigman', env.ENV, currentBuild.currentResult, 'app-sigman', 'https://github.com/peopulse/app-sigman')
        }
    }
>>>>>>> 339a03e (Addning Jenkins File)
}
