def Get_MDG(project) {
    // split the project.
    def values = project.split('/')
    if ( values.size() > 1 ) {
        return values[1]
    }
    // no prefix, likely just the project name then
    return project
}

def project_checkout() {
    // checkout the project
    git url: "https://osm.etsi.org/gerrit/${env.GERRIT_PROJECT}"

    sh "git fetch origin ${env.GERRIT_REFSPEC}"
    if (env.GERRIT_PATCHSET_REVISION.size() > 0 ) {
        sh "git checkout -f ${env.GERRIT_PATCHSET_REVISION}"
    }
}

def devops_checkout() {
    dir('devops') {
        git url: 'https://osm.etsi.org/gerrit/osm/devops'
    }
}

node {
    mdg = Get_MDG("${env.GERRIT_PROJECT}")
    println("MDG is ${mdg}")

    if ( env.GERRIT_EVENT_TYPE.equals('change-merged') ) {
        def downstream_params = [
            string(name: 'GERRIT_BRANCH', value: env.GERRIT_BRANCH),
            string(name: 'GERRIT_PROJECT', value: env.GERRIT_PROJECT),
            string(name: 'GERRIT_REFSPEC', value: env.GERRIT_REFSPEC),
            string(name: 'GERRIT_PATCHSET_REVISION', value: env.GERRIT_PATCHSET_REVISION),
        ]
        result = build job: "${mdg}/${env.GERRIT_BRANCH}", parameters: downstream_params, propagate: true
        if (result.getResult() != 'SUCCESS') {
            project = result.getProjectName()
            build = result.getNumber()
            error("${project} build ${build} failed")
        }
    }
    else {
        stage('Prepare') {
            sh 'env'
            devops_checkout()
        }

        stage('Checkout') {
            project_checkout()
        }

        container_name = "${env.GERRIT_PROJECT}-${env.GERRIT_BRANCH}"


        stage('Docker-Build') {
            sh "docker build -t ${container_name} ."
        }

        withDockerContainer("${container_name}") {
            stage('Docker-Setup') {
                sh '''
                   groupadd -o -g $(id -g) -r jenkins
                   useradd -o -u $(id -u) --create-home -r -g  jenkins jenkins
                   '''
            }
            stage('Test') {
                sh 'devops-stages/stage-test.sh'
            }
            stage('Build') {
                sh 'devops-stages/stage-build.sh'
            }
            stage('Archive') {
                sh 'devops-stages/stage-archive.sh'
                archiveArtifacts artifacts: "dists/**,pool/${mdg}/*.deb", fingerprint: true
            }
        }
     }
}
