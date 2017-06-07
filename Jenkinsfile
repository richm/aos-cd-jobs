#!/usr/bin/env groovy

def mail_success(list,detail) {
    mail(
        to: "${list}",
        from: "aos-cd@redhat.com",
        replyTo: 'jupierce@redhat.com',
        subject: "[aos-devel] Cluster ${OPERATION} complete: ${CLUSTER_NAME}",
        body: """\
${detail}

Jenkins job: ${env.BUILD_URL}
""");
}

node('buildvm-devops') {

    properties(
            [[$class              : 'ParametersDefinitionProperty',
              parameterDefinitions:
                      [
                              [$class: 'hudson.model.StringParameterDefinition', defaultValue: 'aos-devel@redhat.com, aos-qe@redhat.com', description: 'Success Mailing List', name: 'MAIL_LIST_SUCCESS'],
                              [$class: 'hudson.model.StringParameterDefinition', defaultValue: 'jupierce@redhat.com, mwoodson@redhat.com', description: 'Success for minor cluster operation', name: 'MAIL_LIST_SUCCESS_MINOR'],
                              [$class: 'hudson.model.StringParameterDefinition', defaultValue: 'jupierce@redhat.com, mwoodson@redhat.com', description: 'Failure Mailing List', name: 'MAIL_LIST_FAILURE'],
                              [$class: 'hudson.model.ChoiceParameterDefinition', choices: "test-key\ncicd\ndev-preview-int\ndev-preview-stg\npreview\nfree-int\nfree-stg\nstarter-us-east-1\nstarter-us-east-2\nstarter-us-west-2", name: 'CLUSTER_NAME', description: 'The name of the cluster to affect'],
                              [$class: 'hudson.model.ChoiceParameterDefinition', choices: "noop\nreinstall\ndelete\ninstall\nupgrade", name: 'OPERATION', description: 'Operation to perform'],
                              [$class: 'hudson.model.ChoiceParameterDefinition', choices: "interactive\nquiet\nsilent\nautomatic", name: 'MODE', description: 'Select automatic to prevent input prompt. Select quiet to prevent aos-devel emails. Select silent to prevent any success email.'],
                      ]
             ]]
    )

    // Force Jenkins to fail early if this is the first time this job has been run/and or new parameters have not been discovered.
    echo "MAIL_LIST_SUCCESS:[${MAIL_LIST_SUCCESS}], MAIL_LIST_FAILURE:[${MAIL_LIST_FAILURE}], CLUSTER_NAME:${CLUSTER_NAME}, OPERATION:${OPERATION}, MODE:${MODE}"

    currentBuild.displayName = "#${currentBuild.number} - ${OPERATION} ${CLUSTER_NAME}"
    
    if ( MODE != "automatic" ) {
        input "Are you certain you want to =====>${OPERATION}<===== the =====>${CLUSTER_NAME}<===== cluster?"
    }

    if ( OPERATION == "noop" ) {
        error( "The default operation (noop) is invalid. An operation must be explicitly specified." )
    }

    // Clusters that can be deleted & installed
    disposableCluster = [ 'test-key', 'cicd', 'dev-preview-int'].contains( CLUSTER_NAME )

    if ( OPERATION != "upgrade" && !disposableCluster ) {
        error( "This script is not permitted to perform that operation" )
    }

    try {   
        cluster_detail = ""

        stage( 'Cluster operation' ) {
            sshagent([CLUSTER_NAME]) {
                
                echo "Initial cluster state:"
                sh "ssh -o StrictHostKeyChecking=no opsmedic@use-tower2.ops.rhcloud.com status"
                echo "\n\n"
                
                if ( OPERATION == "reinstall" ) {
                    sh "ssh -o StrictHostKeyChecking=no opsmedic@use-tower2.ops.rhcloud.com delete"
                    sh "ssh -o StrictHostKeyChecking=no opsmedic@use-tower2.ops.rhcloud.com install"
                } else {
                    sh "ssh -o StrictHostKeyChecking=no opsmedic@use-tower2.ops.rhcloud.com ${OPERATION}"
                }
                if ( OPERATION != "delete" ) {
                    cluster_detail = sh(returnStdout: true, script: "ssh -o StrictHostKeyChecking=no opsmedic@use-tower2.ops.rhcloud.com status").trim()
                    echo "Gathered status:\n${cluster_detail}"
                }
            }
        }

        minorUpdate = [ 'test-key', 'cicd' ].contains( CLUSTER_NAME ) || OPERATION == "delete" || MODE == "quiet"
        
        if ( MODE != "silent" ) {
            // Replace flow control with: https://jenkins.io/blog/2016/12/19/declarative-pipeline-beta/ when available
            mail_success(minorUpdate?MAIL_LIST_SUCCESS_MINOR:MAIL_LIST_SUCCESS, cluster_detail)
        }

    } catch ( err ) {
        // Replace flow control with: https://jenkins.io/blog/2016/12/19/declarative-pipeline-beta/ when available
        mail(to: "${MAIL_LIST_FAILURE}",
                from: "aos-cd@redhat.com",
                subject: "Error during ${OPERATION} on cluster ${CLUSTER_NAME}",
                body: """Encountered an error: ${err}

Jenkins job: ${env.BUILD_URL}
""");
            // Re-throw the error in order to fail the job
            throw err
    }


    if ( OPERATION == "install" || OPERATION == "reinstall" || OPERATION == "upgrade" ) {
        try {
            // Send out a CI message for QE
            build job: 'cluster%2Fsend-ci-msg',
                propagate: false,
                parameters: [
                    [$class: 'hudson.model.StringParameterValue', name: 'CLUSTER_NAME', value: CLUSTER_NAME],
                ]
        } catch ( err2 ) {
            mail(to: "${MAIL_LIST_FAILURE}",
                    from: "aos-cd@redhat.com",
                    subject: "Error sending CI msg during ${OPERATION} on cluster ${CLUSTER_NAME}",
                    body: """Encountered an error: ${err2}

    Jenkins job: ${env.BUILD_URL}
    """);

        }
    }

    if ( ( CLUSTER_NAME == "free-int" || CLUSTER_NAME =="test-key" ) && ( OPERATION == "install" || OPERATION == "reinstall" || OPERATION == "upgrade" )  ) {
        // Run perf1 test on free-int
        build job: 'cluster%2Fperf',
            propagate: false,
            parameters: [
                [$class: 'hudson.model.StringParameterValue', name: 'CLUSTER_NAME', value: CLUSTER_NAME],
                [$class: 'hudson.model.StringParameterValue', name: 'OPERATION', value: 'perf1'],
                [$class: 'hudson.model.StringParameterValue', name: 'MODE', value: 'automatic'],
            ]
    }


}
