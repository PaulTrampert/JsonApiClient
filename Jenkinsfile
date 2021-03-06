@Library('github-release-helpers@v0.2.1')
def releaseInfo

pipeline {
  agent {
    docker {
      image "mcr.microsoft.com/dotnet/core/sdk:2.2"
      args "-e HOME=$HOME"
    }
  }

  options {
    buildDiscarder(logRotator(numToKeepStr:'5'))
  }

  environment {
		PROJECT_NAME = "PTrampert.JsonApiClient"
	}

  stages {
		stage('Build Release Info') {
      steps {
        script {
          releaseInfo = generateGithubReleaseInfo(
            'PaulTrampert',
            "$PROJECT_NAME",
            'v',
            'Github User/Pass',
            'https://api.github.com',
            BRANCH_NAME == "master" ? null : BRANCH_NAME,
            env.BUILD_NUMBER
          )

          echo "Next version is ${releaseInfo.nextVersion().toString()}."
          echo "Changelog:\n${releaseInfo.changelogToMarkdown()}"
        }
      }
    }

		stage('Test') {
			steps {
				sh "dotnet test ${PROJECT_NAME}.Test/${PROJECT_NAME}.Test.csproj -l trx"
			}

			post {
				always {
					xunit(
            testTimeMargin: '3000',
            thresholdMode: 1,
            thresholds: [
              failed(unstableThreshold: '0')
            ],
            tools: [
              MSTest(
                deleteOutputFiles: true,
                failIfNotNew: true,
                pattern: '**/*.trx',
                skipNoTestFiles: false,
                stopProcessingIfError: true
              )
            ]
          )

          cobertura(
            autoUpdateHealth: false,
            autoUpdateStability: false,
            coberturaReportFile: '**/*.cobertura.xml',
            conditionalCoverageTargets: '70, 0, 0',
            failUnhealthy: false,
            failUnstable: false,
            lineCoverageTargets: '80, 0, 0',
            maxNumberOfBuilds: 0,
            methodCoverageTargets: '80, 0, 0',
            onlyStable: false,
            sourceEncoding: 'ASCII',
            zoomCoverageChart: false
          )
				}
			}
		}

		stage('Package') {
			steps {
				sh "dotnet pack ${PROJECT_NAME}/${PROJECT_NAME}.csproj -c Release /p:Version=${releaseInfo.nextVersion().toString()}"
			}
		}

		stage('Publish Pre-Release') {
      when { expression{env.BRANCH_NAME != 'master'} }
      environment {
        API_KEY = credentials('nexus-nuget-apikey')
      }
      steps {
        sh "dotnet nuget push **/*.nupkg -s 'https://packages.ptrampert.com/repository/nuget-prereleases/' -k ${env.API_KEY}"
      }
    }

		stage('Publish Release') {
      when { expression {env.BRANCH_NAME == 'master'} }
      environment {
        API_KEY = credentials('nuget-api-key')
      }
      steps {
				script {
          publishGithubRelease(
            'PaulTrampert',
            PROJECT_NAME,
            releaseInfo,
            'v',
            'Github User/Pass',
            'https://api.github.com'
          )
        }

        sh "dotnet nuget push **/*.nupkg -s https://api.nuget.org/v3/index.json -k ${env.API_KEY}"
      }
    }
	}

  post {
    failure {
      mail(
        to: 'paul.trampert@ptrampert.com',
        subject: "Build status of ${env.JOB_NAME} changed to ${currentBuild.result}", body: "Build log may be found at ${env.BUILD_URL}"
      )
    }
    always {
      archiveArtifacts '**/*.nupkg'
    }
  }
}