
stage 'Test'
def splits = splitTests parallelism: [$class: 'CountDrivenParallelism', size: 10], generateInclusions: true
def branches = [:]
for (int i = 0; i < splits.size(); i++) {
  def split = splits[i]
  branches["split${i}"] = {
    node('dockerSlave') {
      checkout scm
      writeFile file: (split.includes ? 'inclusions.txt' : 'exclusions.txt'), text: split.list.join("\n")
      writeFile file: (split.includes ? 'exclusions.txt' : 'inclusions.txt'), text: ''
      def mvnHome = tool 'M3'
      //sh "${mvnHome}/bin/mvn -B clean test -Dmaven.test.failure.ignore"
      //step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/*.xml'])
    }
  }
}
parallel branches

node('dockerSlave') {
    def mvnHome = tool 'M3'
    
    sh "rm -rf *"
    sh "rm -rf .git"
    checkout scm
    checkout([$class: 'GitSCM', branches: [[name: '*/master']],
        extensions: [[$class: 'CleanCheckout'],[$class: 'LocalBranch', localBranch: "master"]]])

    stage 'Set Version'
    def originalV = version();
    def major = originalV[0];
    def minor = originalV[1];
    def v = "${major}.${minor}-${env.BUILD_NUMBER}"
    if (v) {
       echo "Building version ${v}"
    }
    sh "${mvnHome}/bin/mvn -B versions:set -DgenerateBackupPoms=false -DnewVersion=${v}"
    sh 'git add .'
    sh "git commit -m 'Raise version'"
    sh "git tag v${v}"

    stage 'Changelog'
    sh "npm install git+ssh://git@git.gentics.com:psc/changelog-generator.git#master -g"
    sh "changelog2html -t template.html  src/main/changelog/ > target/changelog.html"

    stage 'Release Build'
    sshagent(['601b6ce9-37f7-439a-ac0b-8e368947d98d']) {
      sh "${mvnHome}/bin/mvn -B -DskipTests clean deploy"
      sh "git push origin master"
      sh "git push origin v${v}"
    }

    stage 'Docker Build'
    withEnv(['DOCKER_HOST=tcp://gemini.office:2375']) {
        sh "captain build"
        sh "captain push"
    }
}

def version() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)-.*</version>'
    matcher ? matcher[0][1].tokenize(".") : null
}
