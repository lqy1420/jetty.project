#!groovy

def jdks = ["jdk8", "jdk9"]
def oss = ["linux-docker"] //windows?  ,"linux-docker"
def builds = [:]
for (def os in oss) {
  for (def jdk in jdks) {
    builds[os+"_"+jdk] = getFullBuild( jdk, os )
  }
}

parallel builds

def getFullBuild(jdk, os) {
  return {
    node(os) {
      // System Dependent Locations
      def mvntool = tool name: 'maven3', type: 'hudson.tasks.Maven$MavenInstallation'
      def jdktool = tool name: "$jdk", type: 'hudson.model.JDK'

      // Environment
      List mvnEnv = ["PATH+MVN=${mvntool}/bin", "PATH+JDK=${jdktool}/bin", "JAVA_HOME=${jdktool}/", "MAVEN_HOME=${mvntool}"]
      mvnEnv.add("MAVEN_OPTS=-Xms256m -Xmx1024m -Djava.awt.headless=true")

      try
      {
        stage('Checkout') {
          checkout scm
        }
      } catch (Exception e) {
        notifyBuild("Checkout Failure", jdk)
        throw e
      }

      try
      {
        stage('Compile') {
          withEnv(mvnEnv) {
            timeout(time: 15, unit: 'MINUTES') {
              withMaven(
                      maven: 'maven3',
                      jdk: "$jdk",
                      options: disableMvnReporters(),
                      mavenLocalRepo: "${env.JENKINS_HOME}/${env.EXECUTOR_NUMBER}") {
                sh "mvn -V -B clean install -Dtest=None -T6"
              }

            }
          }
        }
      } catch(Exception e) {
        notifyBuild("Compile Failure", jdk)
        throw e
      }

      try
      {
        stage('Javadoc') {
          withEnv(mvnEnv) {
            timeout(time: 20, unit: 'MINUTES') {
              withMaven(
                      maven: 'maven3',
                      jdk: "$jdk",
                      options: disableMvnReporters(),
                      mavenLocalRepo: "${env.JENKINS_HOME}/${env.EXECUTOR_NUMBER}") {
                sh "mvn -V -B javadoc:javadoc -T5"
              }
            }
          }
        }
      } catch(Exception e) {
        notifyBuild("Javadoc Failure", jdk)
        throw e
      }

      try
      {
        stage('Test') {
          withEnv(mvnEnv) {
            timeout(time: 360, unit: 'MINUTES') {
              // Run test phase / ignore test failures
              withMaven(
                      maven: 'maven3',
                      jdk: "$jdk",
                      options: disableMvnReporters(),
                      mavenLocalRepo: "${env.JENKINS_HOME}/${env.EXECUTOR_NUMBER}") {
                //
                sh "mvn -V -B install -Dmaven.test.failure.ignore=true -Prun-its -T3 -e -Dmaven.repo.local=${env.JENKINS_HOME}/${env.EXECUTOR_NUMBER}"
              }
              // withMaven doesn't label..
              // Report failures in the jenkins UI
              junit testResults:'**/target/surefire-reports/TEST-*.xml'
              // Collect up the jacoco execution results
              def jacocoExcludes =
                      // build tools
                      "**/org/eclipse/jetty/ant/**" + ",**/org/eclipse/jetty/maven/**" +
                              ",**/org/eclipse/jetty/jspc/**" +
                              // example code / documentation
                              ",**/org/eclipse/jetty/embedded/**" + ",**/org/eclipse/jetty/asyncrest/**" +
                              ",**/org/eclipse/jetty/demo/**" +
                              // special environments / late integrations
                              ",**/org/eclipse/jetty/gcloud/**" + ",**/org/eclipse/jetty/infinispan/**" +
                              ",**/org/eclipse/jetty/osgi/**" + ",**/org/eclipse/jetty/spring/**" +
                              ",**/org/eclipse/jetty/http/spi/**" +
                              // test classes
                              ",**/org/eclipse/jetty/tests/**" + ",**/org/eclipse/jetty/test/**";
              step( [$class          : 'JacocoPublisher',
                     inclusionPattern: '**/org/eclipse/jetty/**/*.class',
                     exclusionPattern: jacocoExcludes,
                     execPattern     : '**/target/jacoco.exec',
                     classPattern    : '**/target/classes',
                     sourcePattern   : '**/src/main/java'] )
              // Report on Maven and Javadoc warnings
              step( [$class        : 'WarningsPublisher',
                     consoleParsers: [[parserName: 'Maven'],
                                      [parserName: 'JavaDoc'],
                                      [parserName: 'JavaC']]] )
            }
            if(isUnstable())
            {
              notifyBuild("Unstable / Test Errors", jdk)
            }
          }
        }
      } catch(Exception e) {
        notifyBuild("Test Failure", jdk)
        throw e
      }

      try
      {
        stage ('Compact3') {

          dir("aggregates/jetty-all-compact3") {
            withEnv(mvnEnv) {
              withMaven(
                      maven: 'maven3',
                      jdk: "$jdk",
                      options: disableMvnReporters(),
                      mavenLocalRepo: "${env.JENKINS_HOME}/${env.EXECUTOR_NUMBER}") {
                sh "mvn -V -B -Pcompact3 clean install -T5"
              }
            }
          }
        }
      } catch(Exception e) {
        notifyBuild("Compact3 Failure", jdk)
        throw e
      }
    }
  }
}


// True if this build is part of the "active" branches
// for Jetty.
def isActiveBranch()
{
  def branchName = "${env.BRANCH_NAME}"
  return ( branchName == "master" ||
          ( branchName.startsWith("jetty-") && branchName.endsWith(".x") ) );
}

// Test if the Jenkins Pipeline or Step has marked the
// current build as unstable
def isUnstable()
{
  return currentBuild.result == "UNSTABLE"
}

// Send a notification about the build status
def notifyBuild(String buildStatus, String jdk)
{
  if ( !isActiveBranch() )
  {
    // don't send notifications on transient branches
    return
  }

  // default the value
  buildStatus = buildStatus ?: "UNKNOWN"

  def email = "${env.EMAILADDRESS}"
  def summary = "${env.JOB_NAME}#${env.BUILD_NUMBER} - ${buildStatus} with jdk ${jdk}"
  def detail = """<h4>Job: <a href='${env.JOB_URL}'>${env.JOB_NAME}</a> [#${env.BUILD_NUMBER}]</h4>
    <p><b>${buildStatus}</b></p>
    <table>
      <tr><td>Build</td><td><a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></td><tr>
      <tr><td>Console</td><td><a href='${env.BUILD_URL}console'>${env.BUILD_URL}console</a></td><tr>
      <tr><td>Test Report</td><td><a href='${env.BUILD_URL}testReport/'>${env.BUILD_URL}testReport/</a></td><tr>
    </table>
    """

  emailext (
          to: email,
          subject: summary,
          body: detail
  )
}

def disableMvnReporters() {
  return [
          concordionPublisher(disabled: true),
          dependenciesFingerprintPublisher(disabled: true),
          findbugsPublisher(disabled: true),
          artifactsPublisher(disabled: true),
          invokerPublisher(disabled: true),
          jgivenPublisher(disabled: true),
          junitPublisher(disabled: true),
          pipelineGraphPublisher(disabled: true),
          openTasksPublisher(disabled: true)];
}

// vim: et:ts=2:sw=2:ft=groovy