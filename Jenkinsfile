#!/usr/bin/env groovy

dockerfile {
    dockerPush = true
    dockerRepos = ['confluentinc/cp-kafkacat',]
    mvnPhase = 'package'
    mvnSkipDeploy = true
    nodeLabel = 'docker-oraclejdk8-compose-swarm'
    slackChannel = 'clients-eng'
    upstreamProjects = []
    dockerPullDeps = []
    usePackages = true
    cron = '' // Disable the cron because this job requires parameters
}
