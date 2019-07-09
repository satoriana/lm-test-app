#!groovy
// -*- coding: utf-8; mode: Groovy; -*-

properties([
    buildDiscarder (logRotator (artifactDaysToKeepStr: '', artifactNumToKeepStr: '10', daysToKeepStr: '', numToKeepStr: '10')),
    disableConcurrentBuilds (),
])

def werf_run(werfargs) {
    sh """#!/bin/bash -el
    set -o pipefail
    source <(multiwerf use 1.0 alpha)
    werf ${werfargs}""".trim()
}

def git_repo() {
    sh "git config --get remote.origin.url > .git/remote-url"
    return readFile(".git/remote-url").trim()
}


def git_commit() {
  sh "git rev-parse HEAD > .git/commit"
  return readFile(".git/commit").trim()
}

def git_branch() {
    sh "git rev-parse --abbrev-ref HEAD > .git/branch"
    return readFile(".git/branch").trim()
}

def DOCKER_REGISTRY = "nexus.lm-edu.flant.ru"
def DOCKER_REGISTRY_CREDENTIALS = "DOCKER_REGISTRY"
def HV = "hv6"
def kubecfg_file_name = "hv-6-kubecfg"

node ('mfominov') {
    timestamps {
        deleteDir()
        stage('repo checkout') {
            checkout scm
        }
        stage('werf build') {
            withCredentials([usernamePassword(credentialsId: "${DOCKER_REGISTRY_CREDENTIALS}", passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                sh("docker login -u ${USERNAME} -p ${PASSWORD} ${DOCKER_REGISTRY}")
                env.WERF_TAG_GIT_COMMIT = git_commit()
                werf_run("build-and-publish --stages-storage :local --images-repo ${DOCKER_REGISTRY}/${HV}")
            }
        }
        stage('werf_deploy') {
            def BRANCH = git_branch()
            if ( BRANCH == "master" ) {
                echo "no auto deploy for master branch"
            } else {
                env.WERF_TAG_GIT_COMMIT = git_commit()
                configFileProvider([configFile(fileId: "${kubecfg_file_name}", targetLocation: './kubecfg', variable: 'kube_cfg')]) {
                    werf_run("deploy --env ${BRANCH} --stages-storage :local --images-repo ${DOCKER_REGISTRY}/${HV} --kube-config=${kube_cfg}")
                }
            }
        }
    }
}