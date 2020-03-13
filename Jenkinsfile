node {

    properties([
        parameters([
            string(name: 'tag', defaultValue: 'docker.isc.ru.nl/rdr/web/rdr-configurable-content-donders', description: 'Docker tag to create')
        ])
    ])

    try {
        stage('Checkout') {
            checkout scm
       }

        stage('Build') {
            sh 'echo "Git hash: `git rev-parse --verify HEAD` , Build on `date`" > build/build.txt'
            def branch = env.BRANCH_NAME
            def version = branch.equals('release') ? 'release' : 'latest'
            echo "Building branch ${branch} and pushing as ${params.tag}:${version}"
            sh "docker build -f build/docker/Dockerfile -t ${params.tag}:${version} --no-cache --pull=true ."
            sh "docker push ${params.tag}:${version}"
        }

        stage('Deploy') {
            def ccDir = "/data/rdr/rdr-configurable-content-deployment"
            if (env.BRANCH_NAME == 'release') {
                echo "Deploying new configurable content to production"
                sshagent (credentials: ['rdr-jenkins-ssh-credentials']) {
                    sh "ssh dr-prod-portal $ccDir/start-update.sh release"
                }
            } else if (env.BRANCH_NAME == 'release-acc') {
                echo "Deploying new configurable content to acceptance"
                sshagent (credentials: ['rdr-jenkins-ssh-credentials']) {
                    sh "ssh dr-acc-portal $ccDir/start-update.sh latest"
                }
            } else {
                echo "Not deploying non release tags"
            }
        }

    } catch(Exception e) {
        echo "Got exception: " + e.getMessage() + " -> " + e.toString()
        env.BUILD_FAILURE = e.getMessage()
        throw e
    } finally {
        if (env.BRANCH_NAME == 'release') {
            echo "Mailing release job status"
            def mailRecipients = env.rdmDondersContentMail
            def jobName = currentBuild.fullDisplayName

            if (env.BUILD_FAILURE == null) {
                echo "Mailing succes"
                emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                mimeType: 'text/html',
                subject: "[Jenkins] ${jobName}",
                to: "${mailRecipients}",
                replyTo: "${mailRecipients}",
                recipientProviders: [[$class: 'CulpritsRecipientProvider']]
            } else {
                echo "Mailing build failure"
                emailext body: "${jobName} failed: ${env.BUILD_FAILURE}",
                  subject: "[Jenkins] ${jobName}",
                  to: "${mailRecipients}",
                  replyTo: "${mailRecipients}"
            }
        }
    }
}
