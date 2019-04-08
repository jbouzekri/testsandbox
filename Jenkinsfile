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
        ),
        containerTemplate(
            name: 'phplint',
            image: 'phpqa/parallel-lint',
            ttyEnabled: true,
            command: 'cat',
        ),
        containerTemplate(
            name: 'phpcpd',
            image: 'phpqa/phpcpd',
            ttyEnabled: true,
            command: 'cat',
        ),
        containerTemplate(
            name: 'phploc',
            image: 'phpqa/phploc',
            ttyEnabled: true,
            command: 'cat',
        ),
    ]
) {

    node(label) {

        stage('Checkout') {
            container('git') {
                checkout()
            }
        }

        stage('Prepare') {
            container('git') {
                sh 'rm -rf build/api'
                sh 'rm -rf build/coverage'
                sh 'rm -rf build/logs'
                sh 'rm -rf build/pdepend'
                sh 'rm -rf build/phpdox'
                sh 'mkdir -p build/api'
                sh 'mkdir -p build/coverage'
                sh 'mkdir -p build/logs'
                sh 'mkdir -p build/pdepend'
                sh 'mkdir -p build/phpdox'
            }
        }

        stage('Analysis') {
            /*container('phplint') {
                phplint()
            }*/

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
                },
                'phpcpd': {
                    container('phpcpd') {
                        phpcpd()
                    }
                },
                'phploc': {
                    container('phploc') {
                        phploc()
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

def phplint () {
    context = "continuous-integration/jenkins/lint"
    setBuildStatus ("${context}", 'Checking php syntax', 'PENDING')

    try {
        sh "parallel-lint --checkstyle --exclude vendor src/ > build/logs/lint-result.xml"
        sh "cat build/logs/lint-result.xml"
    } catch (err) {
        echo "${err}"
        setBuildStatus ("${context}", 'PHP syntax errors detected', 'FAILURE')
        throw err
    } finally {
        def lint = scanForIssues tool: checkStyle(pattern: 'build/logs/lint-result.xml')
        publishIssues issues: [lint], name: 'PHP Lint'
    }

    setBuildStatus ("${context}", 'PHP syntax OK', 'SUCCESS')
}

def phpcs () {
    context = "continuous-integration/jenkins/checkstyle"
    setBuildStatus ("${context}", 'Checking coding style', 'PENDING')

    try {
        sh "phpcs -v --standard=PSR2 --extensions=php --ignore=vendor/ --report=checkstyle --report-file=build/logs/checkstyle-result.xml src/"
    } catch (err) {
        setBuildStatus ("${context}", 'Some code conventions are broken', 'FAILURE')
        throw err
    } finally {
        def checkstyle = scanForIssues tool: phpCodeSniffer(pattern: 'build/logs/checkstyle-result.xml')
        publishIssues issues: [checkstyle]
    }

    setBuildStatus ("${context}", 'Code conventions OK', 'SUCCESS')
}

def phpmd () {
    try {
        sh "phpmd src/ xml cleancode,codesize,controversial,design,naming,unusedcode --reportfile build/logs/pmd-result.xml"
    } catch (err) {
        // don't stop build for mess detector errors
    }

    def pmd = scanForIssues tool: pmdParser(pattern: 'build/logs/pmd-result.xml')
    publishIssues issues: [pmd]
}

def phpcpd () {
    try {
        sh "phpcpd --log-pmd build/logs/cpd-result.xml --exclude vendor ."
    } catch (err) {
        // don't stop build for copy/paste detection errors
    }

    def cpd = scanForIssues tool: pmdParser(pattern: 'build/logs/cpd-result.xml')
    publishIssues issues: [cpd], name: 'Copy/Paste Detection'
}

def phploc () {
    try {
        sh "phploc --count-tests --log-csv build/logs/phploc.csv --log-xml build/logs/phploc.xml src/"
        sh "cat build/logs/phploc.csv"
    } catch (err) {
        // don't stop build for copy/paste detection errors
    }


    drawPlot (
        'A - Lines of code',
        'Lines of Code',
        'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)'
    )
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

void drawPlot (String title, String yaxis, String exclusionValues) {
    plot csvFileName: "${ROOT_DIR}/plot-${UUID.randomUUID().toString()}.csv",
        csvSeries: [[
            file: 'build/logs/phploc.csv',
            exclusionValues: exclusionValues,
            displayTableFlag: false,
            inclusionFlag: 'INCLUDE_BY_STRING',
            url: ''
        ]],
        group: 'phploc',
        title: title,
        style: 'line',
        exclZero: false,
        keepRecords: true,
        logarithmic: false,
        numBuilds: 100,
        useDescr: false,
        yaxis: yaxis,
        yaxisMaximum: '',
        yaxisMinimum: ''
}