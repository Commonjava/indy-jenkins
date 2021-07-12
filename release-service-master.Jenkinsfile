library identifier: 'c3i@master', changelog: false,
retriever: modernSCM([$class: 'GitSCMSource', remote: 'https://pagure.io/c3i-library.git'])
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
          - name: JAVA_HOME
            value: /usr/lib/jvm/java-11-openjdk
          - name: USER
            value: 'jenkins-k8s-config'
          - name: HOME
            value: /home/jenkins
          - name: JAVA_TOOL_OPTIONS
            value: '-XX:+UnlockExperimentalVMOptions -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx4g'
          - name: MAVEN_OPTS
            value: -Xmx6g -Xms1024m -XX:MaxPermSize=512m -Xss8m
          resources:
            requests:
              memory: 4Gi
              cpu: 2000m
            limits:
              memory: 8Gi
              cpu: 4000m
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
          checkout([$class      : 'GitSCM', branches: [[name: params.SVC_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: params.SVC_NAME]], submoduleCfg: [],
                    userRemoteConfigs: [[url: params.SVC_GIT_REPO]]])

          echo "Prepare the release of ${params.SVC_GIT_REPO} branch: ${params.SVC_GIT_BRANCH}"

          sh """#!/bin/bash
          cd ${params.SVC_NAME}
          git checkout ${params.SVC_GIT_BRANCH}
          gpg --allow-secret-key-import --import /home/jenkins/gnupg_keys/private_key.txt
          """
          env.TEMP_TAG = params.SVC_MAJOR_VERSION + '-jenkins-' + currentBuild.id
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
            dir(params.SVC_NAME){
              env.SVC_PMD_VIOLATION = sh (
                  script: 'mvn -B -s ../settings.xml -Pformatting,cq clean install',
                  returnStatus: true
              ) == 0
              sh """
              mkdir -p /home/jenkins/.m2
              cp ../settings.xml /home/jenkins/.m2/settings.xml
              """
              catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh """
                    git config --global user.email "${params.BOT_EMAIL}"
                    git config --global user.name "${BOT_USERNAME}"
                    git commit -am "Update license header"                #commit nothing when there is no file needs to be modified
                    git push https://${BOT_USERNAME}:${BOT_PASSWORD}@`python3 -c 'print("${params.SVC_GIT_REPO}".split("//")[1])'` --all
                    """
              }
            }
          }
        }
      }
    }
    stage('Build'){
      steps{
        dir(params.SVC_NAME){
          echo "Executing build for : ${params.SVC_GIT_REPO} ${params.SVC_MAJOR_VERSION}"
          sh "mvn -B clean package ${params.BUILD_PARAM}"
        }
      }
    }
    stage('Archive'){
      steps{
        dir(params.SVC_NAME){
          echo "Archive"
          sh """
          cd target/quarkus-app/
          echo UEsFBgAAAAAAAAAAAAAAAAAAAAAAAA== | base64 -d > lib.zip
          zip -q -9 -j lib.zip lib/boot/*
          zip -q -9 -j lib.zip lib/main/*
          echo UEsFBgAAAAAAAAAAAAAAAAAAAAAAAA== | base64 -d > quarkus.zip
          zip -q -9 -j quarkus.zip quarkus/*
          """
          archiveArtifacts artifacts: "target/quarkus-app/lib.zip,target/quarkus-app/quarkus-run.jar,target/quarkus-app/app/*.jar,target/quarkus-app/quarkus.zip", fingerprint: true
        }
      }
    }
    stage('Build container') {
      environment {
        BUILDCONFIG_INSTANCE_ID = "service-temp-${currentBuild.id}-${UUID.randomUUID().toString().substring(0,7)}"
      }
      steps {
        dir(params.SVC_NAME){
          script {
            openshift.withCluster() {
              def artifact="target/quarkus-app/app/*${params.SVC_MAJOR_VERSION}*.jar"
              def artifact_file = sh(script: "ls $artifact", returnStdout: true)?.trim()
              env.JAR_NAME = "${BUILD_URL}artifact/$artifact_file"
              def template = readYaml file: '../openshift/service-container.yaml'
              def processed = openshift.process(template,
                '-p', "NAME=${env.BUILDCONFIG_INSTANCE_ID}",
                '-p', "SVC_IMAGE_TAG=${env.TEMP_TAG}",
                '-p', "SVC_IMAGESTREAM_NAME=${params.SVC_IMAGESTREAM_NAME}",
                '-p', "SVC_IMAGESTREAM_NAMESPACE=${params.SVC_IMAGESTREAM_NAMESPACE}",
                '-p', "BUILD_URL=${env.BUILD_URL}",
                '-p', "APP_JAR_NAME=${env.JAR_NAME}",
              )
              def build = c3i.buildAndWait(script: this, objs: processed)
              echo 'Container build succeeds!'
              def ocpBuild = build.object()
              env.RESULTING_IMAGE_REF = ocpBuild.status.outputDockerImageReference
              env.RESULTING_IMAGE_DIGEST = ocpBuild.status.output.to.imageDigest
              def imagestream = openshift.selector('is', ['app': env.BUILDCONFIG_INSTANCE_ID]).object()
              env.RESULTING_IMAGE_REPO = imagestream.status.dockerImageRepository
              env.RESULTING_TAG = env.TEMP_TAG
            }
          }
        }
      }
    }
    stage('tag and push image to quay'){
      when {
        expression {
          return params.FORCE_PUBLISH_IMAGE == true
        }
      }
      steps{
        script{
          openshift.withCluster(){
            def template = readYaml file: 'openshift/service-quay-template.yaml'
            def processed = openshift.process(template,
              '-p', "SVC_VERSION=${params.SVC_MAJOR_VERSION}",
              '-p', "BUILD_URL=${BUILD_URL}",
              '-p', "APP_JAR_NAME=${env.JAR_NAME}",
              '-p', "QUAY_TAG=${params.QUAY_IMAGE_TAG}"
            )
            def build = c3i.buildAndWait(script: this, objs: processed)
            echo 'Publish build succeeds!'
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
  if (!env.SVC_PMD_VIOLATION){
    body += "\nFound code PMD Violations; please check"
  }
  emailext to: recipient, subject: subject, body: body
}