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
          image: quay.io/kaine/indy-stress-tester:latest
          imagePullPolicy: Always
          tty: true
          env:
          - name: NPMREGISTRY
            value: 'https://repository.engineering.redhat.com/nexus/repository/registry.npmjs.org'
          - name: JAVA_TOOL_OPTIONS
            value: '-XX:+UnlockExperimentalVMOptions -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx4g'
          - name: MAVEN_OPTS
            value: '-Xmx4g -Xms1024m -XX:MaxPermSize=512m -Xss8m'
          - name: USER
            value: 'jenkins-k8s-config'
          - name: IMG_BUILD_HOOKS
            valueFrom:
              secretKeyRef:
                key: img-build-hooks.json
                name: img-build-hooks-secrets
          - name: JAVA_HOME
            value: /usr/lib/jvm/java-11-openjdk
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
    timeout(time: 30, unit: 'MINUTES')
  }
  environment {
    PIPELINE_NAMESPACE = readFile('/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
    PIPELINE_USERNAME = sh(returnStdout: true, script: 'id -un').trim()
    GITHUB_URL='https://github.com/Commonjava/indy'
    TARBALL_URL = "${env.GITHUB_URL}/releases/download/indy-parent-${params.INDY_VERSION}/indy-launcher-${params.INDY_VERSION}-skinny.tar.gz"
    DATA_TARBALL_URL = "${env.GITHUB_URL}/releases/download/indy-parent-${params.INDY_VERSION}/indy-launcher-${params.INDY_VERSION}-data.tar.gz"
  }
  stages {
    stage('git checkout') {
      when{
        expression{
          return params.REDO_RELEASE_PERFORM == true
        }
      }
      steps{
        script{
          checkout([$class      : 'GitSCM', doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'indy']], submoduleCfg: [],
                    userRemoteConfigs: [[url: env.GITHUB_URL]]])

          echo "Prepare the release of ${env.GITHUB_URL} tag: indy-parent-${params.INDY_VERSION}"

          sh """
          cd indy
          git checkout indy-parent-${params.INDY_VERSION}
          gpg --allow-secret-key-import --import /home/jenkins/gnupg_keys/private_key.txt
          """
        }
      }
    }
    stage('Release Respin') {
      when{
        expression{
          return params.REDO_RELEASE_PERFORM == true
        }
      }
      steps {
        script {
          withCredentials([
            usernamePassword(credentialsId:'GitHubAccount', passwordVariable:'BOT_PASSWORD', usernameVariable:'BOT_USERNAME'),
            usernamePassword(credentialsId:'OSS-Nexus-Bot', passwordVariable:'OSS_BOT_PASSWORD', usernameVariable:'OSS_BOT_USERNAME'),
            string(credentialsId: 'gnupg_passphrase', variable: 'PASSPHRASE')
          ]){
            dir('indy'){
              sh """
              mkdir -p /home/jenkins/.m2
              mkdir -p ./target/checkout
              cp toolchains.xml /home/jenkins/.m2/toolchains.xml
              cp ../settings-release.xml /home/jenkins/.m2/settings.xml
              sed -i 's/{{_USERNAME}}/${OSS_BOT_USERNAME}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSWORD}}/${OSS_BOT_PASSWORD}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSPHRASE}}/'${PASSPHRASE}'/g' /home/jenkins/.m2/settings.xml
              sed -i s,git@github.com:Commonjava/indy.git,https://`python3 -c 'print("${env.GITHUB_URL}".split("//")[1])'`,g pom.xml
              sed -i s,https://github.com/Commonjava/indy.git,https://`python3 -c 'print("${env.GITHUB_URL}".split("//")[1])'`,g pom.xml
              echo 'scm.url=scm:git:${env.GITHUB_URL}' > ./release.properties
              echo 'scm.tag=indy-parent-${params.INDY_VERSION}' >> ./release.properties
              mvn --batch-mode release:perform
              """
            }
          }
        }
      }
    }
    stage('Build Quay Image') {
      when{
        expression{
          return params.PUSH_TO_QUAY == true
        }
      }
      steps{
        script{
          openshift.withCluster(){
            def template = readYaml file: 'openshift/indy-quay-template.yaml'
            def processed = openshift.process(template,
              '-p', "TARBALL_URL=${env.TARBALL_URL}",
              '-p', "DATA_TARBALL_URL=${env.DATA_TARBALL_URL}",
              //'-p', "QUAY_TAG=${params.INDY_DEV_IMAGE_TAG}"
              '-p', "QUAY_TAG=${params.INDY_VERSION}"
            )
            def build = c3i.buildAndWait(script: this, objs: processed)
            echo 'Publish build succeeds!'
          }
        }
      }
    }
    stage('Build Openshift ImageStream Image') {
      when{
        expression {
          return params.PUSH_TO_IMAGESTREAM == true
        }
      }
      environment {
        BUILDCONFIG_INSTANCE_ID = "indy-temp-${currentBuild.id}-${UUID.randomUUID().toString().substring(0,7)}"
      }
      steps {
        script {
          openshift.withCluster() {
            def template = readYaml file: 'openshift/indy-container.yaml'
            def processed = openshift.process(template,
              '-p', "NAME=${env.BUILDCONFIG_INSTANCE_ID}",
              '-p', "INDY_GIT_REPO=${env.GITHUB_URL}",
              '-p', "INDY_GIT_BRANCH=release",
              '-p', "INDY_IMAGE_TAG=${params.INDY_VERSION}",
              '-p', "INDY_VERSION=${params.INDY_VERSION}",
              '-p', "INDY_IMAGESTREAM_NAME=${params.INDY_IMAGESTREAM_NAME}",
              '-p', "INDY_IMAGESTREAM_NAMESPACE=${params.INDY_IMAGESTREAM_NAMESPACE}",
              '-p', "TARBALL_URL=${env.TARBALL_URL}",
              '-p', "DATA_TARBALL_URL=${env.DATA_TARBALL_URL}",
            )
            def build = c3i.buildAndWait(script: this, objs: processed)
            echo 'Container build succeeds!'
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
  emailext to: recipient, subject: subject, body: body
}