def label = "testsandbox-${UUID.randomUUID().toString()}"

def getRepoURL() {
    sh "git config --get remote.origin.url > .git/remote-url"
    return readFile(".git/remote-url").trim()
}

def getCommitSha() {
    sh "git rev-parse HEAD > .git/current-commit"
    return readFile(".git/current-commit").trim()
}

def setBuildStatus(String message, String state) {
    repoUrl = getRepoURL()
    commitSha = getCommitSha()

    echo repoUrl
    echo commitSha

    step([
        $class: "GitHubCommitStatusSetter",
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
        commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
    ]);
}

podTemplate(
    label: label,
    containers: [
        containerTemplate(
            name: 'git',
            image: 'alpine/git',
            ttyEnabled: true,
            command: 'cat',
        )
    ]
) {

    node(label) {

        stage('set github commit status') {
            container('git') {
                checkout scm

                // workaround https://issues.jenkins-ci.org/browse/JENKINS-38674
                setBuildStatus("Build complete", "SUCCESS")
            }
        }

    }
}