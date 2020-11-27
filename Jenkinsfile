#!/usr/bin/env groovy

import static groovy.json.JsonOutput.*



/* These are variables that can be used to test an un-released version of the Confluent Platform that resides at
 * a different HTTPS Endpoint other than `https://packages.confluent.io`. You do not need to specify *any* of them
 * for normal testing purposes, and are purely here for Confluent Inc's usage only.
 */

// The version to install, set to the "next" version to test the "next" version.
def confluent_package_version = string(name: 'CONFLUENT_PACKAGE_VERSION',
    defaultValue: '',
    description: 'Confluent Version to install and test (ie: 5.4.1)'
)

// The HTTP(S) endpoint from which to obtain the platform packages
def confluent_common_repository_baseurl = string(name: 'CONFLUENT_PACKAGE_BASEURL',
    defaultValue: '',
    description: 'Packaging Base URL from where to download packages (ie: https://packages.confluent.io)'
)

// Confluent's nightly packages use a different version scheme, so this parameter controls the "suffix" value
// of the packages that are installed.
def confluent_release_quality = choice(name: 'CONFLUENT_RELEASE_QUALITY',
    choices: ['prod', 'snapshot'],
    defaultValue: 'prod',
    description: 'Determines the release extention (suffix) (ie: "prod" for public releases, "snapshot" for nightly builds)',
)

// Parameter for the molecule test scenario to run
def molecule_scenario_name = choice(name: 'SCENARIO_NAME',
    choices: ['rbac-scram-custom-rhel', 'rbac-mtls-provided-ubuntu', 'archive-plain-rhel'],
    defaultValue: 'rbac-scram-custom-rhel',
    description: 'The Ansible Molecule scenario name to run',
)

def config = jobConfig {
    nodeLabel = 'docker-oraclejdk8'
    slackChannel = '#ansible-eng'
    timeoutHours = 4
    runMergeCheck = false
    properties = [parameters([confluent_package_version, confluent_common_repository_baseurl, confluent_release_quality, molecule_scenario_name])]
}

def job = {
    stage('Install Molecule and Latest Ansible') {
        sh '''
            sudo pip install --upgrade 'ansible==2.9.*'
            sudo pip install molecule docker
        '''
    }

        def override_config = [:]

        // ansible_fqdn within certs does not match the FQDN that zookeeper verifies
        override_config['zookeeper_custom_java_args'] = '-Dzookeeper.ssl.hostnameVerification=false -Dzookeeper.ssl.quorum.hostnameVerification=false'

        def branch_name = targetBranch().toString()

        if(params.CONFLUENT_PACKAGE_BASEURL) {
            override_config['confluent_common_repository_baseurl'] = params.CONFLUENT_PACKAGE_BASEURL
        }

        if(params.CONFLUENT_PACKAGE_VERSION) {
            override_config['confluent_package_version'] = params.CONFLUENT_PACKAGE_VERSION
            override_config['confluent_repo_version'] = params.CONFLUENT_PACKAGE_VERSION.tokenize('.')[0..1].join('.')

            if(params.CONFLUENT_RELEASE_QUALITY != 'prod') {
                // 'prod' case doesn't need anything overriden
                switch(params.CONFLUENT_RELEASE_QUALITY) {
                    case "snapshot":
                        override_config['confluent_package_redhat_suffix'] = "-${params.CONFLUENT_PACKAGE_VERSION}-0.1.SNAPSHOT"
                        override_config['confluent_package_debian_suffix'] = "=${params.CONFLUENT_PACKAGE_VERSION}~SNAPSHOT-1"

                        // Disable reporting for nightly builds
                        config.testbreakReporting = false
                        config.slackChannel = null
                    break
                    default:
                        error("Unknown release quality ${params.CONFLUENT_RELEASE_QUALITY}")
                    break
                }
            }

    
    stage("Test Scenario: rbac-scram-custom-rhel"){
       withDockerServer([uri: dockerHost()]) {
   
        sh """
        docker rmi molecule_local/geerlingguy/docker-centos7-ansible || true

        cd roles/confluent.test
        molecule ${molecule_args} test -s rbac-scram-custom-rhel
                """
            }
        }
    }

def post = {
    withDockerServer([uri: dockerHost()]) {
        stage("Destroy Scenario: rbac-scram-custom-rhel") {
            sh """
cd roles/confluent.test
molecule destroy -s rbac-scram-custom-rhel || true
"""
        }
    }
 }
}

runJob config, job, post