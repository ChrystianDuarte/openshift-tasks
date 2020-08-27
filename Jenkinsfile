def mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"
def DEV_PROJECT="dev-opentlc"
def STAGE_PROJECT="stage-opentlc"
def ENABLE_QUAY="true"


 pipeline {
   agent {
     label 'maven'
   }
   stages {
     stage('Build App') {
       steps {
         sh "${mvnCmd} install -DskipTests=true"
       }
     }
     stage('Test') {
       steps {
         sh "${mvnCmd} test"
         step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
       }
     }
     stage('Code Analysis') {
       steps {
         script {
           sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
         }
       }
     }
     stage('Archive App') {
       steps {
         sh "${mvnCmd} deploy -DskipTests=true -P nexus3"
       }
     }
     stage('Build Image') {
       steps {
         sh "cp target/openshift-tasks.war target/ROOT.war"
         script {
           openshift.withCluster() {
             openshift.withProject("${DEV_PROJECT}") {
               openshift.selector("bc", "tasks").startBuild("--from-file=target/ROOT.war", "--wait=true")
             }
           }
         }
       }
     }
     stage('Deploy DEV') {
       steps {
         script {
           openshift.withCluster() {
             openshift.withProject(${DEV_PROJECT}) {
               openshift.selector("dc", "tasks").rollout().latest();
             }
           }
         }
       }
     }
     stage('Promote to STAGE?') {
       agent {
         label 'skopeo'
       }
       steps {
         timeout(time:15, unit:'MINUTES') {
             input message: "Promote to STAGE?", ok: "Promote"
         }

         script {
           openshift.withCluster() {
             if (${ENABLE_QUAY}.toBoolean()) {
               withCredentials([usernamePassword(credentialsId: "${openshift.project()}-quay-cicd-secret", usernameVariable: "QUAY_USER", passwordVariable: "QUAY_PWD")]) {
                 sh "skopeo copy docker://quay.io/cduarter/tasks-app:latest docker://quay.io/cduarter/tasks-app:stage --src-creds \"$QUAY_USER:$QUAY_PWD\" --dest-creds \"$QUAY_USER:$QUAY_PWD\" --src-tls-verify=false --dest-tls-verify=false"
               }
             } else {
               openshift.tag("${DEV_PROJECT}/tasks:latest", "${STAGE_PROJECT}/tasks:stage")
             }
           }
         }
       }
     }
     stage('Deploy STAGE') {
       steps {
         script {
           openshift.withCluster() {
             openshift.withProject(${STAGE_PROJECT}) {
               openshift.selector("dc", "tasks").rollout().latest();
             }
           }
         }
       }
     }
   }
 }


