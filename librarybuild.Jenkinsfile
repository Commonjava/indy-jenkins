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
          image: registry.redhat.io/openshift3/jenkins-agent-maven-35-rhel7:v3.11.219-1
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
          mv ./settings.xml /home/jenkins/.m2/settings.xml
          """
          checkout([$class      : 'GitSCM', branches: [[name: params.LIB_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: params.LIB_NAME], [$class: 'CleanCheckout']],
                    submoduleCfg: [], userRemoteConfigs: [[url: params.LIB_GIT_REPO]]])

          env.PR_NO = getPrNo(params.LIB_GIT_BRANCH)
          env.TEMP_TAG = params.LIB_MAJOR_VERSION + '-jenkins-' + currentBuild.id
        }
      }
    }
    stage('Get Version'){
      when {
        expression {
          return params.LIB_GIT_BRANCH == 'release'
        }
      }
      steps{
        sh """# /bin/bash
        echo 'Executing build for : ${params.LIB_GIT_REPO} ${params.LIB_MAJOR_VERSION}'
        curl -X POST "http://indy-infra-nos-automation.cloud.paas.psi.redhat.com/api/admin/stores/maven/hosted" -H "accept: application/json" -H "Content-Type: application/json" -d "{ \\"key\\": \\"maven:hosted:${params.LIB_NAME}-${params.LIB_MAJOR_VERSION}-jenkins-${env.BUILD_NUMBER}\\", \\"disabled\\": false, \\"doctype\\": \\"hosted\\", \\"name\\": \\"${params.LIB_NAME}-${params.LIB_MAJOR_VERSION}-jenkins-${env.BUILD_NUMBER}\\", \\"allow_releases\\": true}"
        sed -i 's/{{_BUILD_ID}}/${params.LIB_NAME}-${params.LIB_MAJOR_VERSION}-jenkins-${env.BUILD_NUMBER}/g' /home/jenkins/.m2/settings.xml
        cd ${params.LIB_NAME}
        mvn versions:set -DnewVersion=${params.LIB_MAJOR_VERSION}
        """
      }
    }
    stage('Build'){
      steps{
        dir(params.LIB_NAME){
          sh "mvn -B -V clean verify"
        }
      }
    }
    stage('Functional test'){
      steps {
        dir(params.LIB_NAME){
          sh 'mvn -B -V verify -Prun-its -Pci'
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
        archiveArtifacts artifacts: "**/*${params.LIB_MAJOR_VERSION}*", fingerprint: true
      }
    }
    stage('Deploy') {
      steps {
        dir(params.LIB_NAME){
          echo "Deploy"
          sh 'mvn help:effective-settings -B -V -DskipTests=true deploy -e'
          if (params.LIB_GIT_BRANCH == 'release'){
              sh """
              curl -X POST "http://indy-infra-nos-automation.cloud.paas.psi.redhat.com/api/promotion/paths/promote" -H "accept: application/json" -H "Content-Type: application/json" -d "{\\"source\\": \\"maven:hosted:${params.LIB_NAME}-${params.LIB_MAJOR_VERSION}-jenkins-${env.BUILD_NUMBER}\\", \\"target\\": \\"maven:hosted:local-deployments\\"}"
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
        if (params.LIB_GIT_BRANCH == 'release'){
          try{
            sh """
            curl -X DELETE "http://indy-infra-nos-automation.cloud.paas.psi.redhat.com/api/admin/stores/maven/hosted/${params.LIB_NAME}-${params.LIB_MAJOR_VERSION}-jenkins-${env.BUILD_NUMBER}" -H "accept: application/json"
            """
          }catch(e){
            echo "Error teardown hosted repo"
          }
        }
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