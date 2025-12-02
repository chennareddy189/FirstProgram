pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_NAME', defaultValue: 'firstprogram', description: 'Docker image name')
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
    booleanParam(name: 'PUSH_TO_REGISTRY', defaultValue: false, description: 'Push image to a registry?')
    string(name: 'REGISTRY', defaultValue: '', description: 'Registry (e.g. localhost:5000) if pushing')
  }

  // helper to run shell or bat depending on agent OS
  environment {
    // no-op placeholder; kept for clarity
  }

  stages {
    stage('Prepare') {
      steps {
        script {
          // small helper closure to execute commands cross-platform
          def runCmd = { cmd ->
            if (isUnix()) {
              sh cmd
            } else {
              bat "@echo off && ${cmd}"
            }
          }
          // expose to the script scope
          env.RUN_CMD = "true"
          // stash the closure for use in later stages via a binding
          binding.setVariable('runCmd', runCmd)
        }
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        script {
          runCmd('mvn -B -DskipTests clean package')
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def image = "${params.IMAGE_NAME}:${params.IMAGE_TAG}"
          runCmd("docker build -t ${image} .")
          if (params.PUSH_TO_REGISTRY && params.REGISTRY?.trim()) {
            def target = "${params.REGISTRY}/${image}"
            runCmd("docker tag ${image} ${target}")
            runCmd("docker push ${target}")
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        script {
          // Assumes kubectl is configured on the Jenkins agent and has access to the target cluster
          runCmd('kubectl apply -f k8s/namespace.yaml || true')
          runCmd('kubectl apply -f k8s/deployment.yaml')
          runCmd('kubectl apply -f k8s/service.yaml')
        }
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      echo "Pipeline succeeded"
    }
    failure {
      echo "Pipeline failed"
    }
  }
}
