#!groovy
import groovy.json.JsonOutput
import groovy.json.JsonSlurper

def getWorkspaceId(organization, workspace_name) {
    def response = httpRequest(
        customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
        url: "https://app.terraform.io/api/v2/organizations/" + organization + "/workspaces/" + workspace_name
    )

    def data = new JsonSlurper().parseText(response.content)
    if (!data.data.id) {
        println 'cannot find workspace'
        exit 0
    } else {
        println 'found workspace'
        println ('Workspace Id: ' + data.data.id)
        return data.data.id
    }
}

def getWorkspaceVars(workspace_id, var_keys) {
    def response = httpRequest(
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        url: 'https://app.terraform.io/api/v2/workspaces/' + workspace_id + '/vars'
    )

    Map data = new JsonSlurper().parseText(response.content)

    Map toreturn = [:]

    for (item in data.data) {
        if (var_keys.contains(item.attributes.key)) {
            toreturn[item.attributes.key] = item.id
        }
    }
    return toreturn
}

def updateWorkspaceVar(workspace_id, var_id, var_key, var_value) {
    def payload = """
            {
                "data": {
                  "id":"${var_id}",
                  "attributes": {
                    "key":"${var_key}",
                    "value":"${var_value}",
                    "category":"env",
                    "hcl": false,
                    "sensitive": true
                  },
                  "type":"vars"
                }
            }
          """

    def response = httpRequest(
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'PATCH',
        requestBody: "${payload}",
        url: 'https://app.terraform.io/api/v2/workspaces/' + workspace_id + '/vars/' + var_id
    )
}

def uploadConfigurations(workspace_id) {
    def payload = '''
                  {
                      "data": {
                        "type": "configuration-versions",
                        "attributes": {
                          "auto-queue-runs": false
                        }
                      }
                  }
    '''

    def response = httpRequest(
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'POST',
        requestBody: "${payload}",
        url: 'https://app.terraform.io/api/v2/workspaces/' + workspace_id + '/configuration-versions'
    )

    def data = new JsonSlurper().parseText(response.content)
    upload_url = data.data.attributes['upload-url']
    println upload_url
    return upload_url
}


def doRun(workspaceid) {
    def payload = """
{
    "data": {
        "attributes": {
            "is-destroy":false,
            "message": "Triggered run from Jenkins (build #${env.BUILD_TAG})"
        },
        "type":"runs",
        "relationships": {
            "workspace": {
                "data": {
                    "type": "workspaces",
                    "id": "$workspaceid"
                }
            }
        }
    }
}
    """

    def response = httpRequest(
        customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
        httpMode: 'POST',
        requestBody: "${payload}",
        url: "https://app.terraform.io/api/v2/runs"
    )
    def data = new JsonSlurper().parseText(response.content)

    println ("Run id: " + data.data.id)
    println ("Run status" + data.data.attributes.status)

    return data.data.id
}


def listLiveRun(workspaceid) {
    def response = httpRequest(
        customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
        url: "https://app.terraform.io/api/v2/workspaces/${workspaceid}/runs"
    )
    def data = new JsonSlurper().parseText(response.content)

    println ("Number of runs: " + data.meta.pagination.'total-count')
    def result =  data.data
    def live_runs = []
    def i = 0
    result.each {
        def status = it.attributes.status
        if (status == "discarded" || status == "applied" || status == "errored" || status == "canceled" || status == "force_canceled") {
            i = i + 1
        } else {
            println "$it.id : $it.attributes.status"
            live_runs.push(it.id)
        }
    }

    i = live_runs.size()
    println ("number of live runs:" + i)
    return live_runs
}

def waitForPlan(runid) {
  def status =''
  while (status!="errored") {
    status = getPlanStatus(runid)
    println('Status: ' + status)
    // // If a policy requires an override, prompt in the pipeline
    // if (status == 'finished') {
    //   def getPlanResults = getPlanResults(runid)
    //   return getPlanResults
    // }
    switch (status) {
        case 'finished':
          def getPlanResults = getPlanResults(runid)
          return getPlanResults
        case 'errored':
          println "Plan failed"
          return 0
    }

    sleep(2)
  }
}


def getPlanStatus(runid) {
    def result = ''
    def response = httpRequest(
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://app.terraform.io/api/v2/runs/${runid}/plan"
    )
    def data = new JsonSlurper().parseText(response.content)
    result = data.data.attributes.status
    return result
}


def getPlanResults(runid) {
    def result = ''
    def response = httpRequest(
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://app.terraform.io/api/v2/runs/${runid}/plan"
    )
    def data = new JsonSlurper().parseText(response.content)
    def planResults = data.data.attributes."log-read-url"
    return planResults
}

def waitForRun(runid) {
    def count = 0
    while (true) {
        def status = getRunStatus(runid)
        println('Status: ' + status)

        // If a policy requires an override, prompt in the pipeline
        if (status.startsWith('approve_policy')) {
            def override
            try {
                override = input (message: 'Override policy?', 
                    ok: 'Continue', 
                    parameters: [ booleanParam( 
                        defaultValue: false, 
                        description: 'A policy restriction is enforced.Check the box to approve overriding the policy.',
                                      name: 'Override')
                                 ])
            } catch (err) {
                override = false
            }

            // If we're overriding, tell terraform. Otherwise, discard the run
            if (override == true) {
                println('Overriding!')
                def item = status.split(':')[1]
                println ('item is ' + item)
                def overridden = overridePolicy(item)
                if (!overridden) {
                    println('Could not override the policy')
                    discardRun(runid)
                    error('Could not override the Sentinel policy')
                    break
                } 
            } else {
                println('Rejecting!')
                discardRun(runid)
                error('The pipeline failed due to a Sentinel policy restriction.')
                break
            }
        }

        // If we're ready to apply, prompt in the pipeline to do so
        if (status == 'apply_plan') {
            def apply
            try {
                apply = input (message: 'Confirm Apply', ok: 'Continue', 
                    parameters: [booleanParam(defaultValue: false, 
                    description: 'Would you like to continue to apply this run?', name: 'Apply')]) 
            } catch (err) { 
                apply = false 
            }

            // If we're going to apply, tell Terraform. Otherwise, discard the run
            if (apply == true) {
                println('Applying plan')
                applyRun(runid)
                break
            }
            else {
                println('Rejecting!')
                discardRun(runid)
                error('The pipeline failed due to a manual rejection of the plan.')
                break
            }
        }
        if (count > 60) break
        count++
        sleep(2)
    } //while true
}

def getRunStatus(runid) {
  def result = ''
  def response = httpRequest(
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://app.terraform.io/api/v2/runs/${runid}"
    )
  def data = new JsonSlurper().parseText(response.content)
  switch (data.data.attributes.status) {
        case 'pending':
        case 'plan_queued':
      result = 'pending'
      break
        case 'planning':
      result = 'planning'
      break
        case 'planned':
      result = 'planned'
      break
        case 'cost_estimating':
        case 'cost_estimated':
      result = 'costing'
      break
        case 'policy_checking':
      result = 'policy'
      break
        case 'policy_override':
      println(response.content)
      result = 'approve_policy:' + data.data.relationships['policy-checks'].data[0].id
      break
        case 'policy_checked':
      result = 'apply_plan'
      break
        default:
            result = 'running'
      break
  }
  return result
}




def waitForApply(runid) {
    def count = 0
    while (true) {
        def status = getApplyStatus(runid)
        println('Status: ' + status)

        if (status == 'discarded') {
            println('This run has been discarded')
            error('The Terraform run has been discarded, and the pipeline cannot continue.')
            break
        }
        if (status == 'canceled') {
            println('This run has been canceled outside the pipeline')
            error('The Terraform run has been canceled outside the pipeline, and the pipeline cannot continue.')
            break
        }
        if (status == 'errored') {
            println('This run has encountered an error while applying')
            error('The Terraform run has encountered an error while applying, and the pipeline cannot continue.')
            break
        }
        if (status == 'applied') {
            println('This run has finished applying')
            break
        }

        if (count > 60) break
        count++
        sleep(2)
    }
}


def overridePolicy(policyid) {
  def response = httpRequest(
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'POST',
        url: "https://app.terraform.io/api/v2/policy-checks/${policyid}/actions/override"
    )

  def data = new JsonSlurper().parseText(response.content)
  if (data.data.attributes.status != 'overridden') {
    return false
  }
    else {
    return true
    }
}

def getApplyStatus(runid) {
  def result = ''
  def response = httpRequest(
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://app.terraform.io/api/v2/runs/${runid}"
    )
  def data = new JsonSlurper().parseText(response.content)
  result = data.data.attributes.status
  
  return result
}


def applyRun(runid) {
  def response = httpRequest(
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'POST',
        responseBody: '{ comment: "Apply confirmed" }',
        url: "https://app.terraform.io/api/v2/runs/${runid}/actions/apply"
    )
}


def configuration = [vaultUrl: 'http://192.168.1.73:8200',
        vaultCredentialId: 'vault-token-root']

def secrets = [
        [path: 'secret1/innovation-lab', engineVersion: 1, secretValues: [
                [envVar: 'api_token', vaultKey: 'api_token2']]
        ]
]

pipeline {
  agent any
  parameters {
      string(name: 'ORGANIZATION', defaultValue: 'innovation-lab', description: '')
      string(name: 'WORKSPACE_NAME', defaultValue: 'terraform-simple-instance', description: '')
      string(name: 'OWNER', defaultValue: 'lxpocsgv860')
      string(name: 'PREFIX', defaultValue: '172.29.222.84')
  }
  environment {
          AWS_ACCESS_KEY_ID = "AKIAWSEH7GKTNQXQMO4V"
          AWS_SECRET_ACCESS_KEY = "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"
          AWS_REGION = "ap-southeast-1"

          VAULT_ADDR="http://192.168.1.73:8200"
          ROLE_ID="9641db0a-4b4d-576b-71ab-196106a82271"
          SECRET_ID=credentials("SECRET_ID")

          //BEARER_TOKEN = "ECReMbuw02A2cw.atlasv1.JvPqUHbs6OnEkfzq1d4nyXNLVcvTw4O0IPzz2dg89uTz4aFenR1uv8d5E7sQmOktXNc"
          //TF_RUN_ID = "${params.RUN_ID}"
          TF_WORKSPACE_NAME = "${params.WORKSPACE_NAME}"
          TF_ORG_NAME =  "${params.ORGANIZATION}"

    }

    stages {
                stage('declareTokenEnvVar') {
            steps {
                script {
                    // created imperatively, so we can modified & used at later stages
                    env.BEARER_TOKEN = "notatoken"
                }
            }
        }

        stage('assignToken') {
            steps {
                script {
                    withVault([configuration: configuration, vaultSecrets: secrets]) {
                        env.BEARER_TOKEN = env.api_token
                    }
                }
            }
        }

        stage('printToken') {
            steps {
                echo "BEARER_TOKEN=${env.BEARER_TOKEN}"
            }
        }
        
        stage('Get Workspace Id') {
            steps{
                script {
                    echo "BEARER_TOKEN=${env.BEARER_TOKEN}"
                }
                script {
                    env.TF_WORKSPACE_ID =  getWorkspaceId(env.TF_ORG_NAME, env.TF_WORKSPACE_NAME)
                }
                echo "TF_ORG_NAME is ${env.TF_ORG_NAME}"
                echo "TF_WORKSPACE_NAME is ${env.TF_WORKSPACE_NAME}"
                echo "TF_WORKSPACE_ID is ${env.TF_WORKSPACE_ID}"
                echo "AWS_REGION is $AWS_REGION"
            }
        }

        stage('ListRuns in a Workspace') {
            steps{
                script {
                    listLiveRun(env.TF_WORKSPACE_ID)
                }
            }
        }

        stage('Change Variables') {
            steps {
                script {
                    TFE_VARS = getWorkspaceVars(env.TF_WORKSPACE_ID, ['owner', 'prefix'])
                    println TFE_VARS
                }
                script {
                    updateWorkspaceVar(env.TF_WORKSPACE_ID, TFE_VARS.get('owner'), 'owner', params.OWNER)
                    updateWorkspaceVar(env.TF_WORKSPACE_ID, TFE_VARS.get('prefix'), 'prefix', params.PREFIX)
                }
            }
        }

        stage('WorkspaceRun') {
            steps{
                script {
                    PLAN_ID = doRun(env.TF_WORKSPACE_ID)
                }
            }
        }


        stage('WorkspacePlanCheck') {
            steps {
                script {
                    plan_url = waitForPlan(PLAN_ID)
                    println(plan_url)

                    //sh """ #!/bin/bash
                    //    echo 
                    //    echo 'Downloading Plan File for build ${BUILD_NUMBER}'
                    //    wget -O ${BUILD_NUMBER}.txt $plan_url
                    //"""
                }
            }
        }

        stage('WorkspacePolicyCheck') {
            steps {
                script {
                    waitForRun(PLAN_ID)
                }
            }
        }

        stage('WorkspaceApply') {
            steps {
                script {
                    waitForApply(PLAN_ID)
                }
            }
        }
    } //stage
} //pipeline
