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
          image: quay.io/factory2/jenkins-agent:maven-36-rhel7-latest
          imagePullPolicy: Always
          tty: true
          env:
          - name: JAVA_HOME
            value: /usr/lib/jvm/java-11-openjdk
          - name: MAVEN_HOME
            value: /opt/apache-maven-3.6.3
          - name: JAVA_TOOL_OPTIONS
            value: '-XX:+UnlockExperimentalVMOptions -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx4g'
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
        - name: volume-2
          configMap:
            defaultMode: 420
            name: jenkins-openshift-mappings
      """
    }
  }
  stages {
    stage('git checkout') {
      steps{
        script{
          sh """
          mkdir -p /home/jenkins/.m2
          mv ./settings-build.xml /home/jenkins/.m2/settings.xml
          """
          checkout([$class      : 'GitSCM', branches: [[name: params.LIB_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: params.LIB_NAME], [$class: 'CleanCheckout']],
                    submoduleCfg: [], userRemoteConfigs: [[url: params.LIB_GIT_REPO]]])

          env.PR_NO = getPrNo(params.LIB_GIT_BRANCH)
          env.TEMP_TAG = params.LIB_NAME + "-" + params.LIB_GIT_BRANCH + '-jenkins-' + currentBuild.id
        }
      }
    }
    stage('Build'){
      steps{
        dir(params.LIB_NAME){
          sh "$MAVEN_HOME/bin/mvn -B -V clean verify"
        }
      }
    }
    stage('Functional test'){
      steps {
        dir(params.LIB_NAME){
          sh "$MAVEN_HOME/bin/mvn -B -V verify -Prun-its -Pci"
        }
      }
    }
    stage('Archive') {
      when{
        expression {
          return params.LIB_GIT_BRANCH == 'release'
        }
      }
      steps {
        echo "Archive"
        archiveArtifacts artifacts: "**/*${params.LIB_NAME}*", fingerprint: true
      }
    }
    stage('Deploy') {
      steps {
        dir(params.LIB_NAME){
          echo "Deploy"
          sh 'mvn help:effective-settings -B -V -DskipTests=true deploy -e'
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

