pipeline {
  agent {
    kubernetes {
      cloud params.JENKINS_AGENT_CLOUD_NAME
      label "jenkins-slave-${UUID.randomUUID().toString()}"
      serviceAccount "jenkins"
      defaultContainer 'jnlp'
      yaml """
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: "jenkins-${env.JOB_BASE_NAME}"
          indy-pipeline-build-number: "${env.BUILD_NUMBER}"
      spec:
        containers:
        - name: jnlp
          image: quay.io/kaine/indy-stress-tester:latest
          imagePullPolicy: Always
          tty: true
          env:
          - name: USER
            value: 'jenkins-k8s-config'
          - name: HOME
            value: /home/jenkins
          - name: JAVA_TOOL_OPTIONS
            value: '-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx5g'
          - name: MAVEN_OPTS
            value: -Xmx8g -Xms1024m -XX:MaxPermSize=512m -Xss8m
          - name: NPMREGISTRY
            value: 'https://repository.engineering.redhat.com/nexus/repository/registry.npmjs.org'
          resources:
            requests:
              memory: 6Gi
              cpu: 2000m
            limits:
              memory: 6Gi
              cpu: 2000m
          volumeMounts:
          - mountPath: /home/jenkins/sonatype
            name: volume-0
          - mountPath: /home/jenkins/gnupg_keys
            name: volume-1
          - mountPath: /mnt/ocp
            name: volume-2
          workingDir: /home/jenkins
        volumes:
        - name: volume-0
          secret:
            defaultMode: 420
            secretName: sonatype-secrets
        - name: volume-1
          secret:
            defaultMode: 420
            secretName: gnupg
        - name: volume-2
          configMap:
            defaultMode: 420
            name: jenkins-openshift-mappings
      """
    }
  }
  options {
    //timestamps()
    timeout(time: 180, unit: 'MINUTES')
  }
  environment {
    PIPELINE_NAMESPACE = readFile('/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
    PIPELINE_USERNAME = sh(returnStdout: true, script: 'id -un').trim()
  }
  stages {
    stage('git checkout') {
      steps{
        script{
          checkout([$class      : 'GitSCM', branches: [[name: params.INDY_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'indy']], submoduleCfg: [],
                    userRemoteConfigs: [[url: params.INDY_GIT_REPO]]])

          echo "Prepare the release of ${params.INDY_GIT_REPO} branch: ${params.INDY_GIT_BRANCH}"

          sh """
          cd indy
          git checkout ${params.INDY_GIT_BRANCH}
          gpg --allow-secret-key-import --import /home/jenkins/gnupg_keys/private_key.txt
          """
        }
      }
    }
    stage('Verify with same setting'){
      when{
        expression{
          return params.SKIP_VERIFICATION != true
        }
      }
      steps{
        build job: 'indy-playground', propagate: true, wait: true, parameters: [string(name: 'INDY_GIT_BRANCH', value: "${params.INDY_GIT_BRANCH}"),
        string(name: 'INDY_GIT_REPO', value: "${params.INDY_GIT_REPO}"),
        string(name: 'INDY_MAJOR_VERSION', value: "${params.INDY_MAJOR_VERSION}"),
        string(name: 'INDY_DEV_IMAGE_TAG', value: "latest"),
        booleanParam(name: 'TAG_INTO_IMAGESTREAM', value: true),
        booleanParam(name: 'INDY_PREPARE_RELEASE', value: true),
        booleanParam(name: 'FUNCTIONAL_TEST', value: true),
        booleanParam(name: 'STRESS_TEST', value: false),
        string(name: 'QUAY_IMAGE_TAG', value: 'latest')
        ]
      }
    }
    stage('Setup and Formating'){
      steps{
        script{
          withCredentials([
            usernamePassword(credentialsId:'GitHub_Bot', passwordVariable:'BOT_PASSWORD', usernameVariable:'BOT_USERNAME'),
            usernamePassword(credentialsId:'OSS-Nexus-Bot', passwordVariable:'OSS_BOT_PASSWORD', usernameVariable:'OSS_BOT_USERNAME'),
            string(credentialsId: 'gnupg_passphrase', variable: 'PASSPHRASE')
          ]) {
            dir('indy'){
              env.INDY_PMD_VIOLATION = sh (
                  script: 'mvn -B -s ../settings.xml -Pformatting,cq install -Dpmd.skip=true',
                  returnStatus: true
              ) == 0
              sh """
              mkdir -p /home/jenkins/.m2
              cp ../settings-release.xml /home/jenkins/.m2/settings.xml
              sed -i 's/{{_USERNAME}}/${OSS_BOT_USERNAME}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSWORD}}/${OSS_BOT_PASSWORD}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSPHRASE}}/'${PASSPHRASE}'/g' /home/jenkins/.m2/settings.xml
              sed -i s,git@github.com:Commonjava/indy.git,https://`python3 -c 'print("${params.INDY_GIT_REPO}".split("//")[1])'`,g pom.xml
              sed -i s,https://github.com/Commonjava/indy.git,https://`python3 -c 'print("${params.INDY_GIT_REPO}".split("//")[1])'`,g pom.xml
              """
              catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh """
                      git config --global user.email "${params.BOT_EMAIL}"
                      git config --global user.name "${BOT_USERNAME}"
                      git commit -am "Update license header"                #commit nothing when there is no file needs to be modified
                      git push https://${BOT_USERNAME}:${BOT_PASSWORD}@`python3 -c 'print("${params.INDY_GIT_REPO}".split("//")[1])'` --all
                    """
              }
            }
          }
        }
      }
    }
    stage('Release Prepare'){
      steps{
        script{
          withCredentials([
            usernamePassword(credentialsId:'GitHub_Bot', passwordVariable:'BOT_PASSWORD', usernameVariable:'BOT_USERNAME')
          ]){
            dir('indy'){
              env.INDY_NEXT_VERSION = params.INDY_DEV_VERSION ? params.INDY_DEV_VERSION : sh (
                script: """ echo ${params.INDY_MAJOR_VERSION} | awk -F. -v OFS=. 'NF==1{print ++\$NF}; NF>1{if(length(\$NF+1)>length(\$NF))\$(NF-1)++; \$NF=sprintf("%0*d", length(\$NF), (\$NF+1)%(10^length(\$NF))); print}' """,
                returnStdout: true
              ).trim()
              echo "Releasing indy:${params.INDY_MAJOR_VERSION} and setup next dev version ${env.INDY_NEXT_VERSION}-SNAPSHOT"
              sh """
              mvn help:effective-settings
              mvn --batch-mode release:prepare -DreleaseVersion=${params.INDY_MAJOR_VERSION} -DdevelopmentVersion=${env.INDY_NEXT_VERSION}-SNAPSHOT -Dtag=indy-parent-${params.INDY_MAJOR_VERSION} -Dusername=${BOT_USERNAME} -Dpassword=${BOT_PASSWORD}
              """
            }
          }
        }
      }
    }
    stage('Release performing'){
      steps{
        script{
          dir('indy'){
            sh """
            mvn --batch-mode release:perform
            """
          }
        }
      }
    }
    stage('clean up and prepare folder again'){
      steps{
        script{withCredentials([
            usernamePassword(credentialsId:'GitHub_Bot', passwordVariable:'BOT_PASSWORD', usernameVariable:'BOT_USERNAME'),
            usernamePassword(credentialsId:'OSS-Nexus-Bot', passwordVariable:'OSS_BOT_PASSWORD', usernameVariable:'OSS_BOT_USERNAME'),
            string(credentialsId: 'gnupg_passphrase', variable: 'PASSPHRASE')
          ]) {
            dir('indy'){
              sh """
              git reset --hard
              git clean -f -d
              git pull origin ${params.INDY_GIT_BRANCH}
              git checkout release
              git merge -X theirs -m "Merge branch ${params.INDY_GIT_BRANCH} into release" indy-parent-${params.INDY_MAJOR_VERSION}
              git push https://${BOT_USERNAME}:${BOT_PASSWORD}@`python3 -c 'print("${params.INDY_GIT_REPO}".split("//")[1])'` --all
              """
            }
          }
        }
      }
    }
    stage('Build'){
      steps{
        dir('indy'){
          echo "Executing build for : ${params.INDY_GIT_REPO} ${params.INDY_MAJOR_VERSION}"
          sh "mvn -B -V clean verify -DskipNpmConfig=false"
        }
      }
    }
    stage('tag and push image to quay'){
      steps{
        script{
          openshift.withCluster(){
            def artifact="indy/deployments/launcher/target/*-skinny.tar.gz"
            def artifact_file = sh(script: "ls $artifact", returnStdout: true)?.trim()
            env.TARBALL_URL = "${BUILD_URL}artifact/$artifact_file"

            def data_artifact="indy/deployments/launcher/target/*-data.tar.gz"
            def data_artifact_file = sh(script: "ls $data_artifact", returnStdout: true)?.trim()
            env.DATA_TARBALL_URL = "${BUILD_URL}artifact/$data_artifact_file"

            def template = readYaml file: 'openshift/indy-quay-template.yaml'
            def processed = openshift.process(template,
              '-p', "TARBALL_URL=${env.TARBALL_URL}",
              '-p', "DATA_TARBALL_URL=${env.DATA_TARBALL_URL}",
              //'-p', "QUAY_TAG=${params.INDY_DEV_IMAGE_TAG}"
              '-p', "QUAY_TAG=latest-release"
            )
            def build = c3i.buildAndWait(script: this, objs: processed)
            echo 'Publish build succeeds!'
          }
        }
      }
    }
    stage('Archive release artifact') {
      steps {
        archiveArtifacts artifacts: "indy/deployments/launcher/target/*.tar.gz", fingerprint: true
      }
    }
  }
  post {
    success {
      script {
        echo "SUCCEED"
      }
    }
    failure {
      script {
        dir('indy'){
          sh 'git reset --hard'
          env.INDY_SNAPSHOT_DEPENDENCY = sh (
                  script: 'mvn -s ../settings.xml dependency:tree -Dincludes=:::*-SNAPSHOT | grep SNAPSHOT | grep -v -e Downloaded -e Progress -e Downloading -e indy -e Indy -e "\\[ERROR\\]" -e "\\[\\ pom\\ \\]" -e "\\[\\ jar\\ \\]"',
                  returnStdout: true
            ).trim()
          if (params.MAIL_ADDRESS){
            try {
              sendBuildStatusEmail('failed')
            } catch (e) {
              echo "Error sending email: ${e}"
            }
          }
        }
      }
    }
  }
}

def sendBuildStatusEmail(String status) {
  def recipient = params.MAIL_ADDRESS
  def subject = "Jenkins job ${env.JOB_NAME} #${env.BUILD_NUMBER} ${status}."
  def body = "Build URL: ${env.BUILD_URL}"
  if (env.INDY_SNAPSHOT_DEPENDENCY) {
    body += "\nFound snapshot release depenency: \n ${INDY_SNAPSHOT_DEPENDENCY}"
  }
  if (!env.INDY_PMD_VIOLATION){
    body += "\nFound code PMD Violations; please check"
  }
  if (env.PR_NO) {
    subject = "Jenkins job ${env.JOB_NAME}, PR #${env.PR_NO} ${status}."
    body += "\nPull Request: ${env.PR_URL}"
  }
  emailext to: recipient, subject: subject, body: body
}