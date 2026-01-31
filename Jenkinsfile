node {

    def tag, dockerHubUser, containerName, httpPort = ""

    stage('Prepare Environment') {
        echo 'Initialize Environment'
        tag = "3.0"
        withCredentials([usernamePassword(credentialsId: 'dockerHubAccount', usernameVariable: 'dockerUser', passwordVariable: 'dockerPassword')]) {
            dockerHubUser = "$dockerUser"
        }
        containerName = "bankingapp"
        httpPort = "8989"
    }

    stage('Code Checkout') {
        try {
            checkout scm
        }
        catch (Exception e) {
            echo 'Exception occured in Git Code Checkout Stage'
            currentBuild.result = "FAILURE"
        }
    }

    stage('Maven Build') {
        sh "mvn clean package"
    }

    stage('Docker Image Build') {
        echo 'Creating Docker image'
        sh "docker build -t $dockerHubUser/$containerName:$tag --pull --no-cache ."
    }

    /* ===================== SBOM STAGES START ===================== */

    stage('Generate SBOM') {
        echo 'Generating SBOM using Trivy container'
        sh """
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v \$WORKSPACE:/work \
            aquasec/trivy:latest image \
            --format cyclonedx \
            --output /work/sbom.json \
            $dockerHubUser/$containerName:$tag
        """
    }

    stage('Scan SBOM (Security Gate)') {
        echo 'Scanning SBOM for HIGH and CRITICAL vulnerabilities'
        sh """
          docker run --rm \
            -v \$WORKSPACE:/work \
            aquasec/trivy:latest sbom \
            /work/sbom.json \
            --severity HIGH,CRITICAL \
            --exit-code 1
        """
    }

    stage('Archive SBOM') {
        archiveArtifacts artifacts: 'sbom.json', fingerprint: true
    }

    /* ===================== SBOM STAGES END ===================== */

    stage('Publishing Image to DockerHub') {
        echo 'Pushing the docker image to DockerHub'
        withCredentials([usernamePassword(credentialsId: 'dockerHubAccount', usernameVariable: 'dockerUser', passwordVariable: 'dockerPassword')]) {
            sh "docker login -u $dockerUser -p $dockerPassword"
            sh "docker push $dockerUser/$containerName:$tag"
            echo "Image push complete"
        }
    }

    stage('Ansible Playbook Execution') {
        withCredentials([string(credentialsId: 'ssh_password', variable: 'password')]) {
            sh """
              export ANSIBLE_HOST_KEY_CHECKING=False &&
              ansible-playbook -i inventory.yaml containerDeploy.yaml \
              -e httpPort=$httpPort \
              -e containerName=$containerName \
              -e dockerImageTag=$dockerHubUser/$containerName:$tag \
              -e ansible_password=$password \
              -e key_pair_path=/var/lib/jenkins/server.pem \
              --become
            """
        }
    }
}
