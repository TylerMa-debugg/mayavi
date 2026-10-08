pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: sonar-scanner
      image: sonarsource/sonar-scanner-cli:12.2
      command: ["sleep"]
      args: ["infinity"]
      env:
        - name: SONAR_SCANNER_OPTS
          value: "-Xmx2g"
      resources:
        requests:
          cpu: "1"
          memory: 2Gi
        limits:
          memory: 3Gi
    - name: tools
      image: alpine/k8s:1.35.9
      command: ["sleep"]
      args: ["infinity"]
      volumeMounts:
        - name: hadoop-kubeconfig
          mountPath: /hadoop-kube
          readOnly: true
  volumes:
    - name: hadoop-kubeconfig
      secret:
        secretName: hadoop-kubeconfig
        optional: true
'''
        }
    }

    parameters {
        string(name: 'GATE_SEVERITIES', defaultValue: 'BLOCKER,CRITICAL,MAJOR', description: 'SonarQube issue severities that prevent the Hadoop job from running')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    environment {
        SONAR_PROJECT_KEY = 'mayavi'
        GATE = "${params.GATE_SEVERITIES ?: 'BLOCKER,CRITICAL,MAJOR'}"
        HADOOP_NAMESPACE = 'hadoop'
        HADOOP_MASTER_POD = 'hadoop-master-0'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv('sonarqube') {
                        sh 'sonar-scanner -Dsonar.projectKey=$SONAR_PROJECT_KEY -Dsonar.projectName=mayavi -Dsonar.sources=. -Dsonar.python.version=3 -Dsonar.sourceEncoding=UTF-8'
                    }
                }
            }
        }

        stage('Wait for SonarQube Report') {
            steps {
                container('tools') {
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            TASK_URL=$(grep '^ceTaskUrl=' .scannerwork/report-task.txt | cut -d= -f2-)
                            while true; do
                              STATUS=$(curl -s -u "$SONAR_AUTH_TOKEN:" "$TASK_URL" | jq -r .task.status)
                              echo "SonarQube background task status: $STATUS"
                              case "$STATUS" in
                                SUCCESS) break ;;
                                FAILED|CANCELED) exit 1 ;;
                              esac
                              sleep 5
                            done
                        '''
                    }
                }
            }
        }

        stage('Issue Gate') {
            steps {
                container('tools') {
                    withSonarQubeEnv('sonarqube') {
                        script {
                            sh '''
                                curl -s -u "$SONAR_AUTH_TOKEN:" "$SONAR_HOST_URL/api/issues/search?componentKeys=$SONAR_PROJECT_KEY&resolved=false&ps=1&facets=severities" > all-issues.json
                                curl -s -u "$SONAR_AUTH_TOKEN:" "$SONAR_HOST_URL/api/issues/search?componentKeys=$SONAR_PROJECT_KEY&resolved=false&ps=1&severities=$GATE" > gate-issues.json
                                echo "Open issues by severity:"
                                jq -r '.facets[] | select(.property == "severities") | .values[] | "  " + .val + ": " + (.count | tostring)' all-issues.json
                            '''
                            def blocking = sh(returnStdout: true, script: "jq -r .total gate-issues.json").trim()
                            env.BLOCKING_ISSUES = blocking
                            currentBuild.description = "${env.GATE} issues: ${blocking}"
                            if (blocking == '0') {
                                echo "No ${env.GATE} issues found. The Hadoop line count job will run."
                            } else {
                                echo "Found ${blocking} issues with severity in ${env.GATE}. The Hadoop line count job is skipped."
                            }
                        }
                    }
                }
            }
        }

        stage('Hadoop Line Count') {
            when {
                expression { env.BLOCKING_ISSUES == '0' }
            }
            steps {
                container('tools') {
                    sh '''
                        set -o pipefail
                        tar -czf /tmp/repository.tar.gz --exclude=.git --exclude=.scannerwork .
                        kubectl --kubeconfig /hadoop-kube/config -n "$HADOOP_NAMESPACE" exec -i "$HADOOP_MASTER_POD" -c resourcemanager -- /opt/linecount/run-linecount.sh mayavi < /tmp/repository.tar.gz | tee linecount-results.txt
                    '''
                    archiveArtifacts artifacts: 'linecount-results.txt'
                }
            }
        }
    }
}
