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
    git url: "https://osm.etsi.org/gerrit/${GERRIT_PROJECT}"

    sh "git fetch origin ${GERRIT_REFSPEC}"
    if (GERRIT_PATCHSET_REVISION.size() > 0 ) {
        sh "git checkout -f ${GERRIT_PATCHSET_REVISION}"
    }
}

def devops_checkout() {
    dir('devops') {
        git url: 'https://osm.etsi.org/gerrit/osm/devops'
    }
}

node {
    stage('Prepare') {
        sh 'env'
        devops_checkout()
    }

    stage('Checkout') {
        project_checkout()
    }

    container_name = "${GERRIT_PROJECT}-${GERRIT_BRANCH}"

    mdg = Get_MDG("${GERRIT_PROJECT}")
    println("MDG is ${mdg}")

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