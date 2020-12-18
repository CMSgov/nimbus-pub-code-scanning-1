pipeline {
    agent {
    kubernetes {
        yaml '''
              apiVersion: v1
              kind: Pod
              spec:
                containers:
                  - name: codeql
                    image: "artifactory.cms.gov/nimbus-jenkins-core-docker-local/centos-codeql:latest"
                    tty: true
                    command: ["tail", "-f", "/dev/null"]
                    imagePullPolicy: Always
             '''
             }
         }
    environment {
      JFROG_CLI_HOME_DIR = "${env.WORKSPACE}/.jfrog"
      JFROG_CLI_OFFER_CONFIG=false
      GIT_COMMIT = "${env.GIT_COMMIT}"
      }

  options { timestamps() }

  stages {

    stage('Git Checkout') {
      steps {
          container('codeql'){
             sh"""
             git clone --branch ${BRANCH_NAME} https://github.com/CMSgov/nimbus-pub-code-scanning-1.git
             """
          }
        }
      }
    stage('Initializing codeql') {
      steps {
          container('codeql'){
             sh"""
             cd nimbus-pub-code-scanning-1
             codeql-runner-linux init --languages java --config-file .github/codeql/codeql-config.yml --codeql-path /opt/codeql/codeql --repository CMSgov/nimbus-pub-code-scanning-1 --github-url https://github.com --github-auth ${AUTH}
             """
          }
        }
      }
    stage('monitor and build') {
      steps {
          container('codeql'){
             sh"""
             cd nimbus-pub-code-scanning-1
             chmod +x ${env.WORKSPACE}/nimbus-pub-code-scanning-1/codeql-runner/codeql-env.sh
             . ${env.WORKSPACE}/nimbus-pub-code-scanning-1/codeql-runner/codeql-env.sh
             codeql-runner-linux autobuild --language java
             """
          }
        }
      }
    stage('analyze and upload result to github') {
      steps {
          container('codeql'){
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'CBC-DevSecOps', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
             sh"""
             cd nimbus-pub-code-scanning-1
             mkdir -p /tmp/scan-results
             codeql-runner-linux analyze --repository CMSgov/nimbus-pub-code-scanning-1 --github-url https://github.com --commit ${GIT_COMMIT}  --github-auth ${AUTH} --output-dir /tmp/scan-results --ref refs/heads/${BRANCH_NAME}
             aws s3 cp /tmp/scan-results/ s3://nimbus-code-scanning-results/master --recursive
             set +e
             cat /tmp/scan-results/java-builtin.sarif | jq -e '.runs[0].results | select(length > 0)'
             if [ "\$?" -eq 0 ]; then exit 1; fi
             """
            }
          }
        }
      }
  }
}
