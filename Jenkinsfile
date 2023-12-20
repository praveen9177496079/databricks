// Filename: Jenkinsfile
node {
  def GITREPOREMOTE = "https://github.com/praveen9177496079/databricks.git"
  def GITBRANCH     = "main"
  def DBCLIPATH     = "/usr/local/bin/"
  def JQPATH        = "/usr/bin/"
  def JOBPREFIX     = "jenkins-demo"
  def BUNDLETARGET  = "dev"

  stage('Checkout') {
    git branch: GITBRANCH, url: GITREPOREMOTE
  }
  stage('Validate Bundle') {
    sh """#!/bin/bash
          ${DBCLIPATH}/databricks bundle validate -t ${BUNDLETARGET}
       """
  }
  stage('Deploy Bundle') {
    sh """#!/bin/bash
          ${DBCLIPATH}/databricks bundle deploy -t ${BUNDLETARGET}
       """
  }
  stage('Run Unit Tests') {
    sh """#!/bin/bash
          ${DBCLIPATH}/databricks bundle run -t ${BUNDLETARGET} run-unit-tests
       """
  }
  stage('Run Notebook') {
    sh """#!/bin/bash
          ${DBCLIPATH}/databricks bundle run -t ${BUNDLETARGET} run-dabdemo-notebook
       """
  }
  stage('Evaluate Notebook Runs') {
    sh """#!/bin/bash
          ${DBCLIPATH}/databricks bundle run -t ${BUNDLETARGET} evaluate-notebook-runs
       """
  }
  stage('Import Test Results') {
    def DATABRICKS_BUNDLE_WORKSPACE_ROOT_PATH
    def getPath = "${DBCLIPATH}/databricks bundle validate -t ${BUNDLETARGET} | ${JQPATH}/jq -r .workspace.file_path"
    def output = sh(script: getPath, returnStdout: true).trim()

    if (output) {
      DATABRICKS_BUNDLE_WORKSPACE_ROOT_PATH = "${output}"
    } else {
      error "Failed to capture output or command execution failed: ${getPath}"
    }

    sh """#!/bin/bash
          ${DBCLIPATH}/databricks workspace export-dir \
          ${DATABRICKS_BUNDLE_WORKSPACE_ROOT_PATH}/Validation/reports/junit/test-reports \
          ${WORKSPACE}/Validation/reports/junit/test-reports \
          -t ${BUNDLETARGET} \
          --overwrite
       """
  }
  stage('Publish Test Results') {
    junit allowEmptyResults: true, testResults: '**/test-results/*.xml', skipPublishingChecks: true
  }
}
