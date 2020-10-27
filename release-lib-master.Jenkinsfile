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
            value: '-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx4g'
          - name: MAVEN_OPTS
            value: -Xmx8g -Xms1024m -XX:MaxPermSize=512m -Xss8m
          resources:
            requests:
              memory: 4Gi
              cpu: 2000m
            limits:
              memory: 4Gi
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
    timeout(time: 120, unit: 'MINUTES')
  }
  environment {
    PIPELINE_NAMESPACE = readFile('/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
    PIPELINE_USERNAME = sh(returnStdout: true, script: 'id -un').trim()
  }
  stages {
    stage('git checkout') {
      steps{
        script{
          checkout([$class      : 'GitSCM', branches: [[name: params.LIB_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: params.LIB_NAME]], submoduleCfg: [],
                    userRemoteConfigs: [[url: params.LIB_GIT_REPO]]])

          echo "Prepare the release of ${params.LIB_GIT_REPO} branch: ${params.LIB_GIT_BRANCH}"

          sh """#!/bin/bash
          cd ${params.LIB_NAME}
          git checkout ${params.LIB_GIT_BRANCH}
          gpg --allow-secret-key-import --import /home/jenkins/gnupg_keys/private_key.txt
          """
        }
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
            dir(params.LIB_NAME){
              env.LIB_PMD_VIOLATION = sh (
                  script: 'mvn -B -s ../settings.xml -Pformatting,cq clean install',
                  returnStatus: true
              ) == 0
              sh """
              mkdir -p /home/jenkins/.m2
              cp ../settings-release.xml /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSPHRASE}}/${PASSPHRASE}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_USERNAME}}/${OSS_BOT_USERNAME}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSWORD}}/${OSS_BOT_PASSWORD}/g' /home/jenkins/.m2/settings.xml
              sed -i s,git@github.com:Commonjava/${params.LIB_NAME}.git,https://`python3 -c 'print("${params.LIB_GIT_REPO}".split("//")[1])'`,g pom.xml
              sed -i s,https://github.com/Commonjava/${params.LIB_NAME}.git,https://`python3 -c 'print("${params.LIB_GIT_REPO}".split("//")[1])'`,g pom.xml
              """
              catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh """
                    git config --global user.email "${params.BOT_EMAIL}"
                    git config --global user.name "${BOT_USERNAME}"
                    git commit -am "Update license header"                #commit nothing when there is no file needs to be modified
                    git push https://${BOT_USERNAME}:${BOT_PASSWORD}@`python3 -c 'print("${params.LIB_GIT_REPO}".split("//")[1])'` --all
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
            dir(params.LIB_NAME){
              env.LIB_NEXT_VERSION = params.LIB_DEV_VERSION ? params.LIB_DEV_VERSION : sh (
                script: """ echo ${params.LIB_MAJOR_VERSION} | awk -F. -v OFS=. 'NF==1{print ++\$NF}; NF>1{if(length(\$NF+1)>length(\$NF))\$(NF-1)++; \$NF=sprintf("%0*d", length(\$NF), (\$NF+1)%(10^length(\$NF))); print}' """,
                returnStdout: true
              ).trim()
              echo "Releasing ${params.LIB_NAME}:${params.LIB_MAJOR_VERSION} and setup next dev version ${env.LIB_NEXT_VERSION}-SNAPSHOT"
              sh """
              mvn help:effective-settings
              mvn --batch-mode release:prepare -DreleaseVersion=${params.LIB_MAJOR_VERSION} -DdevelopmentVersion=${env.LIB_NEXT_VERSION}-SNAPSHOT -Dtag=${params.LIB_NAME}-${params.LIB_MAJOR_VERSION} -Dusername=${BOT_USERNAME} -Dpassword=${BOT_PASSWORD}
              """
            }
          }
        }
      }
    }
    stage('Release performing'){
      steps{
        script{
          dir(params.LIB_NAME){
            sh """
            mvn --batch-mode release:perform
            """
          }
        }
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

def sendBuildStatusEmail(String status) {
  def recipient = params.MAIL_ADDRESS
  def subject = "Jenkins job ${env.JOB_NAME} #${env.BUILD_NUMBER} ${status}."
  def body = "Build URL: ${env.BUILD_URL}"
  if (env.PR_NO) {
    subject = "Jenkins job ${env.JOB_NAME}, PR #${env.PR_NO} ${status}."
    body += "\nPull Request: ${env.PR_URL}"
  }
  if (!env.INDY_PMD_VIOLATION){
    body += "\nFound code PMD Violations; please check"
  }
  emailext to: recipient, subject: subject, body: body
}