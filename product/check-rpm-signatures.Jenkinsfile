#!/usr/bin/env groovy

// PARAMETERS for this pipeline:
    // none

def MIDSTM_BRANCH = "crw-2.6-rhel-8" // redhat-developer/codeready-workspaces branch to use as source of the new branches

def buildNode = "rhel7-releng||rhel7-32gb||rhel7-16gb||rhel7-8gb" // node label
timeout(120) {
    node("${buildNode}"){ 
        stage("Create branches") {
            wrap([$class: 'TimestamperBuildWrapper']) {
                sh('curl -sSLO https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/'+ MIDSTM_BRANCH + '/product/util.groovy')
                def util = load "${WORKSPACE}/util.groovy"
                cleanWs()
                CRW_VERSION = util.getCrwVersion(MIDSTM_BRANCH)
                println "CRW_VERSION = '" + CRW_VERSION + "'"
                util.installSkopeo(CRW_VERSION)
                util.installYq()
                sh '''#!/bin/bash -xe
                sudo yum -y install podman; podman --version
                '''
                sh('curl -sSLO https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/'+ MIDSTM_BRANCH + '/product/getLatestImageTags.sh && chmod +x getLatestImageTags.sh')
                sh('curl -sSLO https://raw.githubusercontent.com/redhat-developer/codeready-workspaces/'+ MIDSTM_BRANCH + '/product/check-rpm-signatures.sh && chmod +x check-rpm-signatures.sh')
                withCredentials([string(credentialsId:'devstudio-release.token', variable: 'GITHUB_TOKEN'), file(credentialsId: 'crw-build.keytab', variable: 'CRW_KEYTAB')]) {
                    util.bootstrap(CRW_KEYTAB)
                    currentBuild.description="Checking RPM signatures ..."
                    sh ('''
export KRB5CCNAME=/var/tmp/crw-build_ccache
kinit "crw-build/codeready-workspaces-jenkins.rhev-ci-vms.eng.rdu2.redhat.com@REDHAT.COM" -kt ''' + CRW_KEYTAB + '''
klist # verify working
./check-rpm-signatures.sh''' // set branch if needed with -b ''' + MIDSTM_BRANCH
                    )
                    MISSING_SIGS = sh(script: '''#!/bin/bash -xe
                        if [[ -f missing.signatures.txt ]]; then cat missing.signatures.txt; fi
                    ''', returnStdout: true).trim()
                    if (MISSING_SIGS.equals("")){
                        currentBuild.description="RPM signatures checked: OK"
                    } else {
                        archiveArtifacts fingerprint: true, artifacts:"missing.signatures.txt"
                        currentBuild.description="Unsigned RPM content found!"
                        currentBuild.result = 'FAILED'
                    }
                } //with 
            } // wrap
        } // stage
    } // node 
} // timeout
