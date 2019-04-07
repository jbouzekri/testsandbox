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
        ),
        containerTemplate(
            name: 'phpmd',
            image: 'phpqa/phpmd',
            ttyEnabled: true,
            command: 'cat',
        )
    ]
) {

    node(label) {
        stage('Checkout') {
            container('git') {
                checkout()
            }
        }

        stage('Analysis') {

            parallel(
                'phpcs': {
                    container('phpcs') {
                        phpcs()
                    }
                },
                'phpmd': {
                    container('phpmd') {
                        phpmd()
                    }
                }
            )

        }
    }
}

def checkout () {
    context = "continuous-integration/jenkins/checkout"

    checkout scm

    // workaround https://issues.jenkins-ci.org/browse/JENKINS-38674
    repoUrl = getRepoURL()
    commitSha = getCommitSha()

    echo "Detected repo url : ${repoUrl}"
    echo "Code at commit : ${commitSha}"

    setBuildStatus ("${context}", 'Code checkout OK', 'SUCCESS')
}

def phpcs () {
    context = "continuous-integration/jenkins/checkstyle"
    setBuildStatus ("${context}", 'Checking coding style', 'PENDING')

    try {
        sh "phpcs -v --standard=PSR2 --report=checkstyle --report-file=checkstyle-result.xml src/"
    } catch (err) {
        setBuildStatus ("${context}", 'Some code conventions are broken', 'FAILURE')
        throw err
    } finally {
        def checkstyle = scanForIssues tool: phpCodeSniffer(pattern: 'checkstyle-result.xml')
        publishIssues issues: [checkstyle]
    }

    setBuildStatus ("${context}", 'Code conventions OK', 'SUCCESS')
}

def phpmd () {
    try {
        sh "phpmd src/ xml --reportfile pmd-result.xml"
    } catch (err) {

    } finally {
        def pmd = scanForIssues tool: pmdParser(pattern: 'pmd-result.xml')
        publishIssues issues: [pmd]
    }
}

def getRepoURL () {
    sh "git config --get remote.origin.url > .git/remote-url"
    return readFile(".git/remote-url").trim()
}

def getCommitSha () {
    sh "git rev-parse HEAD > .git/current-commit"
    return readFile(".git/current-commit").trim()
}

void setBuildStatus (String context, String message, String state) {
    step([
        $class: "GitHubCommitStatusSetter",
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
        commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
        contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
    ]);
}