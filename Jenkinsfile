#!/usr/bin/env groovy

dockerfile {
    dockerPush = true
    dockerRepos = ['confluentinc/cp-kcat',]
    mvnPhase = 'package'
    mvnSkipDeploy = true
    nodeLabel = 'docker-debian-jdk8-compose'
    slackChannel = 'clients-eng'
    upstreamProjects = []
    dockerPullDeps = ['confluentinc/cp-base-new']
    usePackages = true
    cron = '' // Disable the cron because this job requires parameters
    cpImages = true
    osTypes = ['ubi9']
    nanoVersion = true
    buildArm = true
}
