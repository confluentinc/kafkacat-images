#!/usr/bin/env groovy

dockerfile {
    dockerPush = true
    dockerRepos = ['confluentinc/cp-kafkacat',]
    mvnPhase = 'package'
    mvnSkipDeploy = true
    nodeLabel = 'docker-oraclejdk8-compose-swarm'
    slackChannel = 'clients-eng'
    upstreamProjects = []
    dockerPullDeps = ['confluentinc/cp-base-new']
    usePackages = true
    cron = '' // Disable the cron because this job requires parameters
    cpImages = true
    osTypes = ['deb8', 'rhel8']
}
