import org.jenkinsci.plugins.pipeline.modeldefinition.Utils
import groovy.transform.Field

def label = "testsandbox-${UUID.randomUUID().toString()}"

@Field
def repoUrl

@Field
def commitSha

@Field
def currentTag

@Field
def currentBranch

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
            name: 'composer',
            image: 'composer',
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
        /*containerTemplate(
            name: 'phpunit',
            image: 'phpunit/phpunit:7.4.0',
            ttyEnabled: true,
            command: 'cat',
        ),*/
        containerTemplate(
            name: 'docker',
            image: 'docker:18',
            ttyEnabled: true,
            command: 'cat',
            /*envVars: [
                secretEnvVar(key: 'REGISTRY_PASSWORD', secretName: 'docker-registry-secret', secretKey: 'password'),
            ],*/
        ),
    ]
) {

    node(label) {

        stage('Checkout') {
            container('git') {
                checkout()
            }
        }

        /*stage('Prepare') {
            container('git') {
                sh 'rm -rf vendor'
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

            container('composer') {
                composer()
            }
        }*/

        //stage('Analysis') {
            /*container('phplint') {
                phplint()
            }*/

            /*parallel(
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
                }*//*,
                'phpunit': {
                    container('phpunit') {
                        phpunit()
                    }
                }*/
            //)
        //}

        conditionnalStage('Build', currentTag) {
            parallel(
                'app1': {
                    container('docker') {
                        echo "Building on docker container !!!!"
                    }
                }
            )
        }

        conditionnalStage('Deploy', currentTag) {
            echo "Deploy in progress ..."
        }
    }
}

def conditionnalStage(name, execute, block) {
    return stage(name, execute ? block : {
        echo "Skipped stage $name"
        Utils.markStageSkippedForConditional(STAGE_NAME)
    })
}

/*def dockerbuild_app1 () {
    def dockerPassword = sh(
        script: 'echo $REGISTRY_PASSWORD',
        returnStdout: true
    ).trim()

    sh "docker login ${dockerRepository} -u ${dockerUser} -p ${dockerPassword}"
    sh "cd api/ && docker build --tag ${dockerRepository}/${dockerWaterMetricsImage}:${version} ."
    sh "docker push ${dockerRepository}/${dockerWaterMetricsImage}:${version}"
}

def dockerbuild (String dockerfile, String imageName, String imageVersion) {

}*/


def checkout () {
    context = "continuous-integration/jenkins/checkout"

    checkout scm

    // workaround https://issues.jenkins-ci.org/browse/JENKINS-38674
    repoUrl = getRepoURL()
    commitSha = getCommitSha()
    currentTag = getCurrentTag()
    currentBranch = getCurrentBranch(currentTag)

    echo "Detected repo url : ${repoUrl}"
    echo "Current branch : ${currentBranch}"
    echo "Code at commit : ${commitSha}"
    if (currentTag) {
        echo "Current tag : ${currentTag}"
    } else {
        echo "No tag detected on current commit"
    }

    setBuildStatus ("${context}", 'Code checkout OK', 'SUCCESS')
}

def composer () {
    context = "continuous-integration/jenkins/composer"
    setBuildStatus ("${context}", 'Running composer install', 'PENDING')

    try {
        sh "composer install"
    } catch (err) {
        echo "${err}"
        setBuildStatus ("${context}", 'Composer install failed', 'FAILURE')
    }

    setBuildStatus ("${context}", 'Composer install done', 'SUCCESS')
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

    def cpd = scanForIssues tool: cpd(pattern: 'build/logs/cpd-result.xml')
    publishIssues issues: [cpd], name: 'Copy/Paste Detection'
}

def phploc () {
    try {
        sh "phploc --count-tests --log-csv build/logs/phploc.csv --log-xml build/logs/phploc.xml src/"
    } catch (err) {
        // don't stop build for copy/paste detection errors
    }

    drawPhplocPlots ()
}

def phpunit () {
    context = "continuous-integration/jenkins/unittest"
    setBuildStatus ("${context}", 'Running unit tests', 'PENDING')

    try {
        sh "vendor/bin/phpunit --coverage-clover build/logs/phpunit-coverage.xml --log-junit build/logs/phpunit-result.xml tests/"
    } catch (err) {
        setBuildStatus ("${context}", 'Unit tests failed', 'FAILURE')
        throw err
    } finally {
        step([
            $class: 'XUnitBuilder',
            thresholds: [[$class: 'FailedThreshold', unstableThreshold: '1']],
            tools: [[$class: 'PHPUnitJunitHudsonTestType', pattern: 'build/logs/phpunit-result.xml']]
        ])

        step([
            $class: 'CloverPublisher',
            cloverReportDir: 'build/logs',
            cloverReportFileName: 'phpunit-coverage.xml'
        ])
    }

    setBuildStatus ("${context}", 'Unit tests OK OK', 'SUCCESS')
}

def getRepoURL () {
    sh "git config --get remote.origin.url > .git/remote-url"
    return readFile(".git/remote-url").trim()
}

def getCommitSha () {
    sh "git rev-parse HEAD > .git/current-commit"
    return readFile(".git/current-commit").trim()
}

def getCurrentTag () {
    // add --tags if you want to consider non annoted tags too
    sh "git describe --exact-match HEAD | head -n 1 > .git/current-tag"
    return readFile(".git/current-tag").trim()
}

def getCurrentBranch (tagValue) {
    if ( tagValue ) {
        sh "git branch --no-merge --contains ${tagValue} > .git/current-branch"

    } else {
        sh "git name-rev --name-only --always HEAD > .git/current-branch"
    }

    return readFile(".git/current-branch").trim()
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

void drawPlot (String title, String yaxis, String exclusionValues, String[] strExclusionSet) {
    step ([
        $class: 'PlotBuilder',
        csvFileName: "plot-${title}.csv",
        csvSeries: [[
            file: 'build/logs/phploc.csv',
            exclusionValues: exclusionValues,
            strExclusionSet: strExclusionSet,
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
        numBuilds: '100',
        useDescr: false,
        yaxis: yaxis,
        yaxisMaximum: '',
        yaxisMinimum: ''
    ]);
}

void drawPhplocPlots () {
    drawPlot (
        'A - Lines of code',
        'Lines of Code',
        'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)',
        (String[]) ['Lines of Code (LOC)', 'Comment Lines of Code (CLOC)', 'Non-Comment Lines of Code (NCLOC)', 'Logical Lines of Code (LLOC)']
    )
    drawPlot (
        'B - Structures Containers',
        'Count',
        'Directories,Files,Namespaces',
        (String[]) ['Namespaces', 'Files', 'Directories']
    )
    drawPlot (
        'C - Average Length',
        'Average Lines of Code',
        'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)',
        (String[]) ['Average Function Length (LLOC)', 'Average Method Length (LLOC)', 'Average Class Length (LLOC)']
    )
    drawPlot (
        'D - Relative Cyclomatic Complexity',
        'Cyclomatic Complexity by Structure',
        'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods',
        (String[]) ['Cyclomatic Complexity / Lines of Code', 'Cyclomatic Complexity / Number of Methods']
    )
    drawPlot (
        'E - Types of Classes',
        'Count',
        'Classes,Abstract Classes,Concrete Classes',
        (String[]) ['Abstract Classes', 'Classes', 'Concrete Classes']
    )
    drawPlot (
        'F - Types of Methods',
        'Count',
        'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods',
        (String[]) ['Methods', 'Static Methods', 'Non-Static Methods', 'Public Methods', 'Non-Public Methods']
    )
    drawPlot (
        'G - Types of Constants',
        'Count',
        'Constants,Global Constants,Class Constants',
        (String[]) ['Class Constants', 'Global Constants', 'Constants']
    )
    drawPlot (
        'H - Types of Functions',
        'Count',
        'Functions,Named Functions,Anonymous Functions',
        (String[]) ['Functions', 'Named Functions', 'Anonymous Functions']
    )
    drawPlot (
        'I - Testing',
        'Count',
        'Test Classes,Test Methods',
        (String[]) ['Test Classes', 'Test Methods']
    )
    drawPlot (
        'AB - Code Structure by Logical Lines of Code',
        'Logical Lines of Code',
        'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes',
        (String[]) ['Classes Length (LLOC)', 'LLOC outside functions or classes', 'Functions Length (LLOC)', 'Logical Lines of Code (LLOC)']
    )
    drawPlot (
        'BB - Structure Objects',
        'Count',
        'Interfaces,Traits,Classes,Methods,Functions,Constants',
        (String[]) ['Functions', 'Traits', 'Classes', 'Methods', 'Interfaces', 'Constants']
    )
}