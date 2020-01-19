library identifier: 'c3i@master', changelog: false,
  retriever: modernSCM([$class: 'GitSCMSource', remote: 'https://pagure.io/c3i-library.git'])
import static org.apache.commons.lang.StringEscapeUtils.escapeHtml;
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
          image: registry.redhat.io/openshift3/jenkins-agent-maven-35-rhel7:v3.11
          imagePullPolicy: Always
          tty: true
          env:
          - name: JAVA_TOOL_OPTIONS
            value: '-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx4g'
          - name: MAVEN_OPTS
            value: '-Xmx8g -Xms1024m -XX:MaxPermSize=512m -Xss8m'
          - name: USER
            value: 'jenkins-k8s-config'
          - name: IMG_BUILD_HOOKS
            valueFrom:
              secretKeyRef:
                key: img-build-hooks.json
                name: img-build-hooks-secrets
          - name: HOME
            value: /home/jenkins
          resources:
            requests:
              memory: 4Gi
              cpu: 2000m
            limits:
              memory: 8Gi
              cpu: 4000m
          volumeMounts:
          - mountPath: /home/jenkins/.m2
            name: volume-1
          - mountPath: /home/jenkins/sonatype
            name: volume-0
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
            secretName: maven-secrets
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
    GITHUB_API_URL='https://api.github.com/repos/Commonjava/indy'
  }
  stages {
    stage('git checkout') {
      steps{
        script{
          checkout([$class      : 'GitSCM', branches: [[name: params.INDY_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'indy'], [$class: 'CleanCheckout']], submoduleCfg: [],
                    userRemoteConfigs: [[url: params.INDY_GIT_REPO, refspec: '+refs/heads/*:refs/remotes/origin/* +refs/pull/*/head:refs/remotes/origin/pull/*/head']]])
          env.INDY_GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()

          echo "Building ${params.INDY_GIT_BRANCH} commit: ${env.INDY_GIT_COMMIT}"

          env.PR_NO = getPrNo(params.INDY_GIT_BRANCH)
          env.TEMP_TAG = params.INDY_MAJOR_VERSION + '-jenkins-' + currentBuild.id
        }
      }
    }
    stage('Get Version'){
      steps{
        sh """#!/bin/bash
        echo 'Executing build for : ${params.INDY_GIT_REPO} ${params.INDY_MAJOR_VERSION}:${BUILD_NUMBER}'
        cd indy
        mvn versions:set -DnewVersion=${params.INDY_MAJOR_VERSION}-rc${BUILD_NUMBER}
        """
      }
    }
    stage('Build'){
      steps{
        dir("indy"){
          sh "mvn -B -V clean verify"
        }
      }
    }
    stage('Function test'){
      steps {
        dir("indy"){
          sh 'mvn -B -V verify -Prun-its -Pci'
        }
      }
    }
    stage('Maven License Format Checking'){
      steps{
        dir("indy"){
          sh 'mvn -B -e license:format'
        }
      }
    }
    stage('Archive') {
      steps {
        archiveArtifacts artifacts: "**/*${params.INDY_MAJOR_VERSION}-rc${BUILD_NUMBER}*", fingerprint: true
      }
    }
    stage('Build container') {
      environment {
        BUILDCONFIG_INSTANCE_ID = "indy-temp-${currentBuild.id}-${UUID.randomUUID().toString().substring(0,7)}"
      }
      steps {
        script {
          openshift.withCluster() {
            def artifact="indy/deployments/launcher/target/*-skinny.tar.gz"
            def artifact_file = sh(script: "ls $artifact", returnStdout: true)?.trim()
            env.TARBALL_URL = "${BUILD_URL}artifact/$artifact_file"

            def data_artifact="indy/deployments/launcher/target/*-data.tar.gz"
            def data_artifact_file = sh(script: "ls $data_artifact", returnStdout: true)?.trim()
            env.DATA_TARBALL_URL = "${BUILD_URL}artifact/$data_artifact_file"

            def template = readYaml file: 'openshift/indy-container.yaml'
            def processed = openshift.process(template,
              '-p', "NAME=${env.BUILDCONFIG_INSTANCE_ID}",
              '-p', "INDY_GIT_REPO=${params.INDY_GIT_REPO}",
              '-p', "INDY_GIT_BRANCH=${env.PR_NO ? params.INDY_GIT_BRANCH : env.INDY_GIT_COMMIT}",
              '-p', "INDY_IMAGE_TAG=${env.TEMP_TAG}",
              '-p', "INDY_VERSION=${params.INDY_MAJOR_VERSION}-rc${BUILD_NUMBER}",
              '-p', "INDY_IMAGESTREAM_NAME=${params.INDY_IMAGESTREAM_NAME}",
              '-p', "INDY_IMAGESTREAM_NAMESPACE=${params.INDY_IMAGESTREAM_NAMESPACE}",
              '-p', "TARBALL_URL=${env.TARBALL_URL}",
              '-p', "DATA_TARBALL_URL=${env.DATA_TARBALL_URL}",
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
      post {
        failure {
          echo "Failed to build container image ${env.TEMP_TAG}."
        }
        cleanup {
          script {
            openshift.withCluster() {
              echo 'Tearing down...'
              openshift.selector('bc', [
                'app': env.BUILDCONFIG_INSTANCE_ID,
                'template': 'indy-container-template',
                ]).delete()
            }
          }
        }
      }
    }
    stage('tag and push image to quay'){
      when {
        expression {
          return params.FORCE_PUBLISH_IMAGE == 'true' || !env.PR_NO
        }
      }
      steps{
        script{
          openshift.withCluster(){
            def template = readYaml file: 'openshift/indy-quay-template.yaml'
            def processed = openshift.process(template,
              '-p', "TARBALL_URL=${env.TARBALL_URL}",
              '-p', "DATA_TARBALL_URL=${env.DATA_TARBALL_URL}",
              '-p', "QUAY_TAG=${params.INDY_DEV_IMAGE_TAG}"
            )
            def build = c3i.buildAndWait(script: this, objs: processed)
            echo 'Publish build succeeds!'
          }
        }
      }
    }
    stage('deploy test environment'){
      when {
        expression {
          return params.FORCE_PUBLISH_IMAGE == 'true' || !env.PR_NO
        }
      }
      steps{
        withCredentials([usernamePassword(credentialsId:'Tower_Auth', passwordVariable:'PASSWORD', usernameVariable:'USERNAME')]) {
          sh """#!/bin/bash
          curl -k -u ${USERNAME}:${PASSWORD} \
          -H 'Content-Type: application/json' \
          --data '{}' \
          -X POST ${params.TOWER_HOST}api/v2/job_templates/850/launch/
          """
        }
      }
    }
    /*stage('integration test'){

    }
    stage('stress test'){

    }*/
    stage('deploy indy artifact'){
      when {
        expression {
          return params.FORCE_PUBLISH_IMAGE == 'true' || !env.PR_NO
        }
      }
      steps {
        dir("indy"){
          sh 'mvn help:effective-settings -B -V -DskipTests=true deploy -e'
        }
      }
    }
    stage('Tag image in ImageStream'){
      when {
        expression {
          return "${params.INDY_DEV_IMAGE_TAG}" && params.TAG_INTO_IMAGESTREAM == "true" && (params.FORCE_PUBLISH_IMAGE == 'true' || !env.PR_NO)
        }
      }
      steps{
        script{
          openshift.withCluster() {
            openshift.withProject("${params.INDY_IMAGESTREAM_NAMESPACE}") {
              def sourceRef = "${params.INDY_IMAGESTREAM_NAME}:${env.RESULTING_TAG}"
              def destRef = "${params.INDY_IMAGESTREAM_NAME}:${params.INDY_DEV_IMAGE_TAG}"
              echo "Tagging ${sourceRef} as ${destRef}"
              openshift.tag("${sourceRef}", "${destRef}")
            }
          }
        }
      }
    }
  }
  post {
    cleanup {
      script{
        if (env.RESULTING_TAG) {
          echo "Removing tag ${env.RESULTING_TAG} from the ImageStream..."
          openshift.withCluster() {
            openshift.withProject("${params.INDY_IMAGESTREAM_NAMESPACE}") {
              openshift.tag("${params.INDY_IMAGESTREAM_NAME}:${env.RESULTING_TAG}",
                "-d")
            }
          }
        }
      }
    }
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
@NonCPS
def getPrNo(branch) {
  def prMatch = branch =~ /^(?:.+\/)?pull\/(\d+)\/head$/
  return prMatch ? prMatch[0][1] : ''
}

def sendBuildStatusEmail(String status) {
  def recipient = params.MAIL_ADDRESS
  def subject = "Jenkins job ${env.JOB_NAME} #${env.BUILD_NUMBER} ${status}."
  def body = "Build URL: ${env.BUILD_URL}"
  if (env.PR_NO) {
    subject = "Jenkins job ${env.JOB_NAME}, PR #${env.PR_NO} ${status}."
    body += "\nPull Request: ${env.PR_URL}"
  }
  emailext to: recipient, subject: subject, body: body
}