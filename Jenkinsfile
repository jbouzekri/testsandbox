def label = "testsandbox-${UUID.randomUUID().toString()}"
def repoUrl
def commitSha

podTemplate(
    label: label,
    containers: [
        containerTemplate(
            name: 'git',
            image: 'alpine/git',
            ttyEnabled: true,
            command: 'cat',
        ),
        containerTemplate(
            name: 'phpcs',
            image: 'herloct/phpcs',
            ttyEnabled: true,
            command: 'cat',
        )
    ]
) {

    node(label) {
        stages {
            container('git') {
                stage('Checkout code') {
                    checkout()
                }
            }

            container('phpcs') {
                sh "phpcs --standard=PSR2 --report=xml --report-file=/tmp/checkstyle-result.xml src/"

                def checkstyle = scanForIssues tool: checkStyle(pattern: '/tmp/checkstyle-result.xml')
                publishIssues issues: [checkstyle], filters: [includePackage('io.jenkins.plugins.analysis.*')]
            }
        }
    }
}

def checkout () {
    context = "continuous-integration/jenkins/checkout"

    checkout scm

    repoUrl = getRepoURL()
    commitSha = getCommitSha()

    echo "Detected repo url : ${repoUrl}"
    echo "Code at commit : ${commitSha}"

    setBuildStatus ("${context}", 'Code checkout OK', 'SUCCESS')
}

def getRepoURL() {
    sh "git config --get remote.origin.url > .git/remote-url"
    return readFile(".git/remote-url").trim()
}

def getCommitSha() {
    sh "git rev-parse HEAD > .git/current-commit"
    return readFile(".git/current-commit").trim()
}

void setBuildStatus(String context, String message, String state) {
    // workaround https://issues.jenkins-ci.org/browse/JENKINS-38674
    step([
        $class: "GitHubCommitStatusSetter",
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
        commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
        contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
    ]);
}