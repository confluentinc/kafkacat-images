#!/usr/bin/env groovy

dockerfile {
    dockerPush = false
    dockerRepos = ['confluentinc/cp-kafkacat',]
    mvnPhase = 'package'
    mvnSkipDeploy = true
    nodeLabel = 'docker-oraclejdk8-compose-swarm'
    slackChannel = 'tools-notifications' //TODO: change to correct team
    upstreamProjects = [] //TODO: after roll out update
    dockerPullDeps = ['confluentinc/cp-base-new']
    usePackages = true
    cron = '' // Disable the cron because this job requires parameters
}
