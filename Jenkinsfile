def defaults = [
    "deployUser"    : "deploy",
    "deployKeys"    : ['deploy_2018_production', 'deploy_2018_staging'],
    "releasesToKeep": 5,
]

def environments = [
    "staging": [
        "deployServer": "193.200.242.59"
    ],
    "master": [
        "deployServer": "193.200.242.58"
    ]
]

/***********************************************************/
def getBuildInstance(variables, environments) {
    if (variables.containsKey('deployableBuildName')) {
        print "Vars already set"
    } else {
        print "Setting up instance variables"

        String repoName             = scm.getUserRemoteConfigs()[0].getUrl().tokenize('/').last().split("\\.")[0]
        String homePath             = "/home/${variables.deployUser}"
        String deployPathBase       = "${homePath}/apps/${repoName}"
        String deployableBuildName  = sh(returnStdout: true, script: 'date "+%Y-%m-%d_%H-%M-%S"').trim()
        String deployPathReleases   = "${deployPathBase}/releases"
        String deployPathCurrent    = "${deployPathBase}/current"
        String deployedReleasePath  = "${deployPathReleases}/${deployableBuildName}"

        def computedVars = [
            "repoName"                      : repoName,
            "homePath"                      : homePath,
            "deployServer"                  : environments[env.BRANCH_NAME]['deployServer'],
            "deployableBuildName"           : deployableBuildName,
            "deployPathBase"                : deployPathBase,
            "deployPathReleases"            : deployPathReleases,
            "deployPathCurrent"             : deployPathCurrent,
            "deployedReleasePath"           : deployedReleasePath,
            "deployPathShared"              : "${deployPathBase}/shared",
            "compressedWorspacePath"        : "${env.WORKSPACE_TMP}/${deployableBuildName}.tar.gz",
            "deployedCompressedWorspacePath": "${deployedReleasePath}/${deployableBuildName}.tar.gz",
            "deployedPm2FilePath"           : "${deployedReleasePath}/pm2.${env.BRANCH_NAME}.json",
            "wasTriggeredByUser"            : currentBuild.getBuildCauses()[0].shortDescription.startsWith('Started by user')

        ]
        variables = variables << computedVars
    }

    return variables
}

def buildInstance // Global variable declaration
/***********************************************************/


pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    buildInstance = getBuildInstance(defaults, environments)
                    sh 'rm -rf .node_modules'
                    sh 'npm i'
                    sh "tar czvf ${buildInstance.compressedWorspacePath} -C ${env.WORKSPACE} ." // compress the build to deploy faster
                }
            }
        }
        stage('Test') {
            steps {
                sh 'echo "Should be testing here..."'
                print currentBuild.getBuildCauses()[0].shortDescription
            }
        }
        stage('Deploy') {
            when {
                expression { buildInstance.wasTriggeredByUser || env.BRANCH_NAME != 'master' }
            }
            steps {
                script {
                    sshagent(buildInstance.deployKeys) {
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} mkdir -p ${buildInstance.deployedReleasePath} ${buildInstance.deployPathShared}/logs" // create folders if needed
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} '([ ! -f ${buildInstance.deployPathCurrent} ] && touch ${buildInstance.deployPathCurrent})'" // shitty workaround to avoid errors on first deploy
                        sh "scp ${buildInstance.compressedWorspacePath} ${buildInstance.deployUser}@${buildInstance.deployServer}:${buildInstance.deployedReleasePath}" // copy the compressed buid into the deploy server
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} tar xzvf ${buildInstance.deployedCompressedWorspacePath} -C ${buildInstance.deployedReleasePath}" // extract the build into the releases folder
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} rm -f ${buildInstance.deployPathBase}/last_release" // remove the last_release reference before making a new one
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} mv ${buildInstance.deployPathCurrent} ${buildInstance.deployPathBase}/last_release" // keep a reference of the last release (fail silently)
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} ln -s ${buildInstance.deployedReleasePath} ${buildInstance.deployPathCurrent}" // make the last deployed release the current one
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} ln -s ${buildInstance.deployPathShared}/logs ${buildInstance.deployedReleasePath}" // create logs link into the newest release
                        sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} '(ls -d ${buildInstance.deployPathReleases}/*|head -n -${buildInstance.releasesToKeep})|xargs --no-run-if-empty rm -rf'" // remove old releases
                    }
                }
            }
        }
        stage('Restart') {
            when {
                expression { buildInstance.wasTriggeredByUser || env.BRANCH_NAME != 'master' }
            }
            steps {
                sh 'echo "Should be restaring here..."'
                sshagent(buildInstance.deployKeys) {
                    sh "ssh ${buildInstance.deployUser}@${buildInstance.deployServer} '(export PATH=\$PATH:/home/deploy/.nvm/current/bin/ && pm2 stop ${buildInstance.deployedPm2FilePath})'"
                }
            }
        }
    }
}
