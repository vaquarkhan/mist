parallel (
    Spark_1_5_2 : {
      test_mist("aws-slave-01","1.5.2")
    },
    Spark_1_6_2 : {
      test_mist("aws-slave-02","1.6.2")
    },
    Spark_2_0_2 : {
      test_mist("aws-slave-03","2.0.2")
    },
    Spark_2_1_0 : {
      test_mist("aws-slave-04","2.1.0")
    }
)

node("aws-slave-04") {
    stage('Fix permissions') {
        fix_permissions()
    }

    stage('Public in Maven') {
        sh "${env.WORKSPACE}/sbt/sbt -DsparkVersion=1.5.2 'set pgpPassphrase := Some(Array())' publishSigned"
        sh "${env.WORKSPACE}/sbt/sbt -DsparkVersion=2.1.0 'set pgpPassphrase := Some(Array())' publishSigned"
        sh "${env.WORKSPACE}/sbt/sbt sonatypeRelease"
    }
}

def test_mist(slaveName,sparkVersion) {
  node(slaveName) {
    wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'XTerm']) {
      try {
        stage('Clone project ' + sparkVersion) {
          checkout scm
        }

        stage('Fix permissions') {
            fix_permissions()
        }

        def tag = sh(returnStdout: true, script: "git tag -l --contains HEAD").trim()

        stage('Build project example for HDFS') {
          echo "Building examples with Spark version: " + sparkVersion
          sh "cd ${env.WORKSPACE} && ${env.WORKSPACE}/sbt/sbt -DsparkVersion=${sparkVersion} 'project examples' package"
        }

        stage('Build and test') {
          echo 'prepare for Mist with Spark version - ' + sparkVersion
          def mosquitto = docker.image('ansi/mosquitto:latest').run()
          def hdfs = docker.image('hydrosphere/hdfs:latest').run(" -v ${env.WORKSPACE}:/usr/share/mist -e SPARK_VERSION=${sparkVersion}","start")
          echo 'Testing Mist with Spark version: ' + sparkVersion
          def mistId = sh(returnStdout: true, script: "docker create -v ${env.WORKSPACE}:/usr/share/mist --link ${mosquitto.id}:mosquitto --link ${hdfs.id}:hdfs hydrosphere/mist:tests-${sparkVersion} tests").trim()
            sh "docker start ${mistId}"
            sh "docker logs -f ${mistId}"

          def checkExitCode = sh(script: "docker inspect -f {{.State.ExitCode}} ${mistId}", returnStdout: true).trim()
          echo "Build flag: ${checkExitCode}"
          if ( checkExitCode == "1" ) {
                sh "docker rm -f ${mistId}"
                echo 'remove containers'
                mosquitto.stop()
                hdfs.stop()
                error("Tests failed")
          }
          sh "docker rm -f ${mistId}"

          echo 'remove containers'
          mosquitto.stop()
          hdfs.stop()
        }

        if (tag.startsWith("v")) {
          stage('Public in DockerHub') {
            build_image(sparkVersion)
          }
        }
      }
      catch (err) {
        currentBuild.result = "FAILURE"
        echo "${err}"
        gitEmail = sh(returnStdout: true, script: "git --no-pager show -s --format='%ae' HEAD").trim()
        mail body: "project build error is here: ${env.BUILD_URL}" ,
        from: 'hydro-support@provectus.com',
        replyTo: 'noreply@provectus.com',
        subject: 'project build failed',
        to: gitEmail
        throw err
      }
    }
  }
}

def build_image(sparkVersion) {
  docker.withRegistry('https://index.docker.io/v1/', '2276974e-852b-45ab-bf14-9136e1b31217') {
    echo 'Building Mist with Spark version: ' + sparkVersion
    def mistImg = docker.build("hydrosphere/mist:${env.BRANCH_NAME}-${sparkVersion}", "--build-arg SPARK_VERSION=${sparkVersion} .")
    echo 'Pushing Mist with Spark version: ' + sparkVersion
    mistImg.push()
  }
}

def fix_permissions(){
    sh "sudo chown -R ubuntu:ubuntu ${env.WORKSPACE}"
}
