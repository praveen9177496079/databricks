# databricks
Databricks CI/CD with Jenkins Pipeline Steps
Install java Ubuntu 22.04
sudo apt-get update
sudo apt-get install openjdk-17-jre -y
java -version
Jenkins (LTS) install on Ubuntu 22.04
Refer: https://www.jenkins.io/doc/book/installing/linux/

  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
jenkins --version
Jenkins add to sudoers group
sudo visudo
For admin persmision add this context to sudoers file

jenkins ALL=(ALL) NOPASSWD: ALL
Install required tools
Databricks CLI install Refer: https://docs.databricks.com/en/dev-tools/cli/install.html

Frist install unzip in Ubuntu 22.04

sudo apt-get install unzip -y
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
Install jq Refer: https://jqlang.github.io/jq/download/
sudo apt-get install jq -y
Install pip & wheel
sudo apt-get update
sudo apt install pip -y
pip install --upgrade wheel
Add global environment variables to Jenkins
To set global environment variables in Jenkins, from your Jenkins Dashboard
Manage Jenkins > System Configuration > System > Global properties > Add Click Add and then enter the environment variable’s Name and Value. Repeat this for each additional environment variable.

DATABRICKS_HOST

DATABRICKS_TOKEN

For token generate in Databrciks console In your Databricks workspace, click your Databricks username in the top bar, and then select User Settings from the drop down.

User Settings > Developer > Access tokens > Manage > Generate new token

Note:
Enter a comment that helps you to identify this token in the future, and change the token’s default lifetime of 90 days. To create a token with no lifetime (not recommended), leave the Lifetime (days) box empty (blank).

Create a Jenkins Pipeline
To create the Jenkins Pipeline in Jenkins

Jenkins Dashboard > New Item > Enter an item name > Pipeline project type icon.

Note: Uncheck the box titled Lightweight checkout, if it is already checked

Sample Jenkins file
// Filename: Jenkinsfile
node {
  def GITREPOREMOTE = "https://github.com/<user-name>/<repo-name>.git"
  def GITBRANCH     = "<release-branch-name>"
  def DBCLIPATH     = "<databricks-cli-installation-path>"
  def JQPATH        = "<jq-installation-path>"
  def JOBPREFIX     = "<job-prefix-name>"
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
          ${DATABRICKS_BUNDLE_WORKSPACE_ROOT_PATH}/Validation/Output/test-results \
          ${WORKSPACE}/Validation/Output/test-results \
          -t ${BUNDLETARGET} \
          --overwrite
       """
  }
  stage('Publish Test Results') {
    junit allowEmptyResults: true, testResults: '**/test-results/*.xml', skipPublishingChecks: true
  }
}
In Jenkins pipeline we need pass Environment Variables value
GITREPOREMOTE: Write Git repository url

GITBRANCH: Write branch name

DBCLIPATH: Write Databricks CLI path lacation

JQPATH: Write jq path lacation

JOBPREFIX: Write the Job prefix name

BUNDLETARGET: Write the Envivronment name Ex.Dev, Stage, & Prof

To check Databricks CLI & jq path

whereis datbricks
whereis jq
Once create jenkins file we create in databricks.yaml and give the some value in that.

Sample databricks.yaml file example:

# Filename: databricks.yml
bundle:
  name: <bundle-name>

variables:
  job_prefix:
    description: A unifying prefix for this bundle's job and task names.
    default: <job-prefix-name>
  spark_version:
    description: The cluster's Spark version ID.
    default: <spark-version-id>
  node_type_id:
    description: The cluster's node type ID.
    default: <cluster-node-type-id>

artifacts:
  dabdemo-wheel:
    type: whl
    path: ./Libraries/python/dabdemo

resources:
  jobs:
    run-unit-tests:
      name: ${var.job_prefix}-run-unit-tests
      tasks:
        - task_key: ${var.job_prefix}-run-unit-tests-task
          new_cluster:
            spark_version: ${var.spark_version}
            node_type_id: ${var.node_type_id}
            num_workers: 1
            spark_env_vars:
              WORKSPACEBUNDLEPATH: ${workspace.root_path}
          notebook_task:
            notebook_path: ./run_unit_tests.py
            source: WORKSPACE
          libraries:
            - pypi:
                package: pytest
    run-dabdemo-notebook:
      name: ${var.job_prefix}-run-dabdemo-notebook
      tasks:
        - task_key: ${var.job_prefix}-run-dabdemo-notebook-task
          new_cluster:
            spark_version: ${var.spark_version}
            node_type_id: ${var.node_type_id}
            num_workers: 1
            spark_env_vars:
              WORKSPACEBUNDLEPATH: ${workspace.root_path}
          notebook_task:
            notebook_path: ./dabdemo_notebook.py
            source: WORKSPACE
          libraries:
            - whl: "/Workspace${workspace.root_path}/files/Libraries/python/dabdemo/dist/dabdemo-0.0.1-py3-none-any.whl"
    evaluate-notebook-runs:
      name: ${var.job_prefix}-evaluate-notebook-runs
      tasks:
        - task_key: ${var.job_prefix}-evaluate-notebook-runs-task
          new_cluster:
            spark_version: ${var.spark_version}
            node_type_id: ${var.node_type_id}
            num_workers: 1
            spark_env_vars:
              WORKSPACEBUNDLEPATH: ${workspace.root_path}
          spark_python_task:
            python_file: ./evaluate_notebook_runs.py
            source: WORKSPACE
          libraries:
            - pypi:
                package: unittest-xml-reporting

targets:
  dev:
    mode: development
    workspace:
      host: <databricks-host>
In databricks.yaml files we pass the Databricks host value same as DATABRICKS_HOST variable value.
