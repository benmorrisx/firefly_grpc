#!groovy
// Library imported for additional extension of the application.
// More info can be found here: https://apollo.gismob/g01207227/byok-jenkins-libraries
@Library('byok-jenkins-libraries') _

// Start of the pipeline
pipeline {
  // Lockdown to run on specific slave
  agent { label "slave-byok-zeus"}

  // Added options for a better debug and look
  options {
    skipDefaultCheckout true
  }

  environment {
    buildType   ='Release'
    CA_NAME     ='.fireflytech.org'
    GOPATH      ='/usr/local/go/dev'
    https_proxy ='192.168.0.1:3128'
    http_proxy  ='192.168.0.1:3128'
  }

  // Start of pipeline stages
  stages {
    // SCM checkout
    // Performed using the library.
    stage("Checkout SCM"){
      steps {
        script {
          sh 'printenv'
        }
        deleteDir()
        checkOut()
      }
    }
    // Pre build step. This would configure OS with environemental variables and execute cmake configuration.
    stage("Pre build action") {
      steps {
        sh 'git clone -b $(curl -L https://grpc.io/release) https://github.com/grpc/grpc'
        sh 'pushd . && cd grpc && git submodule update --init && make -j 4 && sudo make install && popd'
      }
    }
    // Application build
    stage("Run build") {
      steps {
        sh 'rm -rf thirdparty && tar -xvf tools/_deps_linux_x64.tar.gz.dat'
        dir('build') {
            sh 'cmake3 -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=$buildType -DBUILD_DB=ON -DCA_NAME="$HA_CA_NAME" -DSERVER_HOST_NAME=$HOSTNAME -DCLIENT_HOST_NAME=$HOSTNAME ..'
            sh 'cmake3 --build . --config $buildType'
        }
      }
    }
    // Docker - server
    stage("Docker - server") {
        steps {
            sh 'docker build --no-cache --network=host --build-arg HTTPS_PROXY=$https_proxy -t  .'
        }
    }
    // Docker - db
    stage("Docker - db"){
        steps {
            sh 'cd db && docker build --build-arg HTTPS_PROXY=$https_proxy -t -db .' 
        }
    }
    // Application test
    stage("Test Application") {
      steps {
        dir('build') {
            script {
                docker.image('-db').withRun('--network=host -p 127.0.0.1:3306:3306') { c ->
                    sh 'sleep 30'
                    sh 'mysql -h 127.0.0.1 -u ha_user --password=Test123!'
                    docker.image('').withRun('--network=host -p 50051:50051 --env db_hostname_env=127.0.0.1') { d ->
                        sh 'sleep 10'
                        sh 'unset http_proxy && unset https_proxy && GRPC_TRACE=all && GRPC_VERBOSITY=DEBUG && ctest -R test_client'
                    }
                }
            }
        }
      }
      post {
        always {
          dir('build/Testing/Temporary') {
            sh 'echo "Printing out logs!"'
            processLogs('LastTest.log')
          }
        }
      }
    }
  }
  // Post pipeline actions.
  // If the job is part of Merge Request add comments to the merge request.
  post {
    failure {
      script {
        if(env.gitlabMergeRequestId)
        {
          withCredentials([string(credentialsId: 'GitLabAPIComment', variable: 'gitLabApiComment')]) {
            commentOnMR("${gitLabApiComment}", '# FAILED', "${env.gitlabMergeRequestTargetProjectId}", "${env.gitlabMergeRequestIid}")
          }
        }
      }
    }
    success {
      script {
        if(env.gitlabMergeRequestId)
        {
          withCredentials([string(credentialsId: 'GitLabAPIComment', variable: 'gitLabApiComment')]) {
            commentOnMR("${gitLabApiComment}", '# PASSED', "${env.gitlabMergeRequestTargetProjectId}", "${env.gitlabMergeRequestIid}")
          }
        }
      }
    }
  }
}
