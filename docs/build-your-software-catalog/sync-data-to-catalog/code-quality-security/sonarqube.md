import Tabs from "@theme/Tabs"
import TabItem from "@theme/TabItem"
import HelmPrerequisites from "../templates/\_ocean_helm_prerequisites_block.mdx"
import HelmParameters from "../templates/\_ocean-advanced-parameters-helm.mdx"
import ResourceMapping from "../templates/\_resource-mapping.mdx"
import DockerParameters from "./\_docker-parameters.mdx"
import SupportedResources from "./\_supported-resources.mdx"

# SonarQube

Our SonarQube integration allows you to import `projects`, `issues` and `analyses` from your SonarQube account into
Port, according to your mapping and definitions.

## Common use cases

- Map `projects`, `issues` and `analyses` in your SonarQube organization environment.
- Watch for object changes (create/update/delete) in real-time, and automatically apply the changes to your entities in
  Port.
- Create/delete SonarQube objects using self-service actions.

## Prerequisites

<HelmPrerequisites />

## Installation

Choose one of the following installation methods:

<Tabs groupId="installation-methods" queryString="installation-methods">

<TabItem value="real-time-always-on" label="Real Time & Always On" default>

Using this installation option means that the integration will be able to update Port in real time using webhooks.

This table summarizes the available parameters for the installation.
Set them as you wish in the script below, then copy it and run it in your terminal:

| Parameter                                | Description                                                                                                                                                                                  | Example                             | Required |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | -------- |
| `port.clientId`                          | Your port client id ([How to get the credentials](https://docs.getport.io/build-your-software-catalog/sync-data-to-catalog/api/#find-your-port-credentials))                                 |                                     | ✅       |
| `port.clientSecret`                      | Your port client secret ([How to get the credentials](https://docs.getport.io/build-your-software-catalog/sync-data-to-catalog/api/#find-your-port-credentials))                             |                                     | ✅       |
| `integration.secrets.sonarApiToken`      | The [SonarQube API token](https://docs.sonarsource.com/sonarqube/9.8/user-guide/user-account/generating-and-using-tokens/#generating-a-token)                                                |                                     | ✅       |
| `integration.config.sonarOrganizationId` | The SonarQube [organization Key](https://docs.sonarsource.com/sonarcloud/appendices/project-information/#project-and-organization-keys) (Not required when using on-prem sonarqube instance) | myOrganization                      | ✅       |
| `integration.config.appHost`             | A URL bounded to the integration container that can be accessed by sonarqube. When used the integration will create webhooks on top of sonarqube to listen to any live changes in the data   | https://my-ocean-integration.com    | ❌       |
| `integration.config.sonarUrl`            | Required if using **On-Prem**, Your SonarQube instance URL                                                                                                                                   | https://my-sonar-cloud-instance.com | ❌       |

<HelmParameters />

<br/>

```bash showLineNumbers
helm repo add --force-update port-labs https://port-labs.github.io/helm-charts
helm upgrade --install my-sonarqube-integration port-labs/port-ocean \
	--set port.clientId="PORT_CLIENT_ID"  \
	--set port.clientSecret="PORT_CLIENT_SECRET"  \
	--set initializePortResources=true  \
	--set scheduledResyncInterval=120  \
	--set integration.identifier="my-sonarqube-integration"  \
	--set integration.type="sonarqube"  \
	--set integration.eventListener.type="POLLING"  \
	--set integration.secrets.sonarApiToken="MY_API_TOKEN"  \
	--set integration.config.sonarOrganizationId="MY_ORG_KEY"
```

</TabItem>

<TabItem value="one-time" label="Scheduled">

  <Tabs groupId="cicd-method" queryString="cicd-method">
  <TabItem value="github" label="GitHub">
This workflow will run the SonarQube integration once and then exit, this is useful for **scheduled** ingestion of data.

:::warning
If you want the integration to update Port in real time using webhooks you should use
the [Real Time & Always On](?installation-methods=real-time-always-on#installation) installation option
:::

Make sure to configure the
following [Github Secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions):

<DockerParameters />

<br/>

Here is an example for `sonarqube-integration.yml` workflow file:

```yaml showLineNumbers
name: SonarQube Exporter Workflow

# This workflow responsible for running SonarQube exporter.

on:
  workflow_dispatch:

jobs:
  run-integration:
    runs-on: ubuntu-latest

    steps:
      - name: Run Sonarqube Integration
        run: |
          # Set Docker image and run the container
          integration_type="sonarqube"
          version="latest"

          image_name="ghcr.io/port-labs/port-ocean-$integration_type:$version"

          docker run -i --rm --platform=linux/amd64 \
          -e OCEAN__EVENT_LISTENER='{"type":"ONCE"}' \
          -e OCEAN__INITIALIZE_PORT_RESOURCES=true \
          -e OCEAN__INTEGRATION__CONFIG__SONAR_API_TOKEN=${{ secrets.OCEAN__INTEGRATION__CONFIG__SONAR_API_TOKEN }} \
          -e OCEAN__INTEGRATION__CONFIG__SONAR_ORGANIZATION_ID=${{ secrets.OCEAN__INTEGRATION__CONFIG__SONAR_ORGANIZATION_ID }} \
          -e OCEAN__INTEGRATION__CONFIG__SONAR_URL=${{ secrets.OCEAN__INTEGRATION__CONFIG__SONAR_URL }} \
          -e OCEAN__PORT__CLIENT_ID=${{ secrets.OCEAN__PORT__CLIENT_ID }} \
          -e OCEAN__PORT__CLIENT_SECRET=${{ secrets.OCEAN__PORT__CLIENT_SECRET }} \
          $image_name

          exit $?
```

  </TabItem>
  <TabItem value="jenkins" label="Jenkins">
This pipeline will run the SonarQube integration once and then exit, this is useful for **scheduled** ingestion of data.

:::tip
Your Jenkins agent should be able to run docker commands.
:::
:::warning
If you want the integration to update Port in real time using webhooks you should use
the [Real Time & Always On](?installation-methods=real-time-always-on#installation) installation option.
:::

Make sure to configure the following [Jenkins Credentials](https://www.jenkins.io/doc/book/using/using-credentials/)
of `Secret Text` type:

<DockerParameters />

<br/>

Here is an example for `Jenkinsfile` groovy pipeline file:

```text showLineNumbers
pipeline {
    agent any

    stages {
        stage('Run SonarQube Integration') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'OCEAN__INTEGRATION__CONFIG__SONAR_API_TOKEN', variable: 'OCEAN__INTEGRATION__CONFIG__SONAR_API_TOKEN'),
                        string(credentialsId: 'OCEAN__INTEGRATION__CONFIG__SONAR_ORGANIZATION_ID', variable: 'OCEAN__INTEGRATION__CONFIG__SONAR_ORGANIZATION_ID'),
                        string(credentialsId: 'OCEAN__PORT__CLIENT_ID', variable: 'OCEAN__PORT__CLIENT_ID'),
                        string(credentialsId: 'OCEAN__PORT__CLIENT_SECRET', variable: 'OCEAN__PORT__CLIENT_SECRET'),
                    ]) {
                        sh('''
                            #Set Docker image and run the container
                            integration_type="sonarqube"
                            version="latest"
                            image_name="ghcr.io/port-labs/port-ocean-${integration_type}:${version}"
                            docker run -i --rm --platform=linux/amd64 \
                                -e OCEAN__EVENT_LISTENER='{"type":"ONCE"}' \
                                -e OCEAN__INITIALIZE_PORT_RESOURCES=true \
                                -e OCEAN__INTEGRATION__CONFIG__SONAR_API_TOKEN=$OCEAN__INTEGRATION__CONFIG__SONAR_API_TOKEN \
                                -e OCEAN__INTEGRATION__CONFIG__SONAR_ORGANIZATION_ID=$OCEAN__INTEGRATION__CONFIG__SONAR_ORGANIZATION_ID \
                                -e OCEAN__PORT__CLIENT_ID=$OCEAN__PORT__CLIENT_ID \
                                -e OCEAN__PORT__CLIENT_SECRET=$OCEAN__PORT__CLIENT_SECRET \
                                $image_name

                            exit $?
                        ''')
                    }
                }
            }
        }
    }
}
```

  </TabItem>
  </Tabs>

</TabItem>

</Tabs>

## Ingesting SonarQube objects

The SonarQube integration uses a YAML configuration to describe the process of loading data into the developer portal.

Here is an example snippet from the config which demonstrates the process for getting `project` data from SonarQube:

```yaml showLineNumbers
resources:
  - kind: projects
    selector:
      query: "true"
    port:
      entity:
        mappings:
          blueprint: '"sonarQubeProject"'
          identifier: .key
          title: .name
          properties:
            organization: .organization
            link: .__link
            lastAnalysisStatus: .__branch.status.qualityGateStatus
            lastAnalysisDate: .__branch.analysisDate
            numberOfBugs: .__measures[]? | select(.metric == "bugs") | .value
            numberOfCodeSmells: .__measures[]? | select(.metric == "code_smells") | .value
            numberOfVulnerabilities: .__measures[]? | select(.metric == "vulnerabilities") | .value
            numberOfHotSpots: .__measures[]? | select(.metric == "security_hotspots") | .value
            numberOfDuplications: .__measures[]? | select(.metric == "duplicated_files") | .value
            coverage: .__measures[]? | select(.metric == "coverage") | .value
            mainBranch: .__branch.name
            tags: .tags
```

The integration makes use of the [JQ JSON processor](https://stedolan.github.io/jq/manual/) to select, modify,
concatenate, transform and perform other operations on existing fields and values from SonarQube's API events.

<ResourceMapping name="SonarQube" category="Code quality & security providers" components={{
SupportedResources: SupportedResources
}}>

```yaml showLineNumbers
resources:
  - kind: projects
    selector:
      query: "true"
    port:
      entity:
        mappings:
          blueprint: '"sonarQubeProject"'
          identifier: .key
          title: .name
          properties:
            organization: .organization
            link: .__link
            lastAnalysisStatus: .__branch.status.qualityGateStatus
            lastAnalysisDate: .__branch.analysisDate
            numberOfBugs: .__measures[]? | select(.metric == "bugs") | .value
            numberOfCodeSmells: .__measures[]? | select(.metric == "code_smells") | .value
            numberOfVulnerabilities: .__measures[]? | select(.metric == "vulnerabilities") | .value
            numberOfHotSpots: .__measures[]? | select(.metric == "security_hotspots") | .value
            numberOfDuplications: .__measures[]? | select(.metric == "duplicated_files") | .value
            coverage: .__measures[]? | select(.metric == "coverage") | .value
            mainBranch: .__branch.name
            tags: .tags
      # highlight-end
  - kind: projects # In this instance project is mapped again with a different filter
    selector:
      query: '.name == "MyProjectName"'
    port:
      entity:
        mappings: ...
```

</ ResourceMapping>

## Examples

Examples of blueprints and the relevant integration configurations:

### Project

<details>
<summary>Projects blueprint</summary>

```json showLineNumbers
{
  "identifier": "sonarQubeProject",
  "title": "SonarQube Project",
  "icon": "sonarqube",
  "schema": {
    "properties": {
      "organization": {
        "type": "string",
        "title": "Organization",
        "icon": "TwoUsers"
      },
      "link": {
        "type": "string",
        "format": "url",
        "title": "Link",
        "icon": "Link"
      },
      "lastAnalysisDate": {
        "type": "string",
        "format": "date-time",
        "icon": "Clock",
        "title": "Last Analysis Date"
      },
      "numberOfBugs": {
        "type": "number",
        "title": "Number Of Bugs"
      },
      "numberOfCodeSmells": {
        "type": "number",
        "title": "Number Of CodeSmells"
      },
      "numberOfVulnerabilities": {
        "type": "number",
        "title": "Number Of Vulnerabilities"
      },
      "numberOfHotSpots": {
        "type": "number",
        "title": "Number Of HotSpots"
      },
      "numberOfDuplications": {
        "type": "number",
        "title": "Number Of Duplications"
      },
      "coverage": {
        "type": "number",
        "title": "Coverage"
      },
      "mainBranch": {
        "type": "string",
        "icon": "Git",
        "title": "Main Branch"
      },
      "tags": {
        "type": "array",
        "title": "Tags"
      }
    },
    "required": []
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {}
}
```

</details>

<details>
<summary>Integration configuration</summary>

```yaml showLineNumbers
createMissingRelatedEntities: true
deleteDependentEntities: true
resources:
  - kind: projects
    selector:
      query: "true"
    port:
      entity:
        mappings:
          blueprint: '"sonarQubeProject"'
          identifier: .key
          title: .name
          properties:
            organization: .organization
            link: .__link
            lastAnalysisStatus: .__branch.status.qualityGateStatus
            lastAnalysisDate: .__branch.analysisDate
            numberOfBugs: .__measures[]? | select(.metric == "bugs") | .value
            numberOfCodeSmells: .__measures[]? | select(.metric == "code_smells") | .value
            numberOfVulnerabilities: .__measures[]? | select(.metric == "vulnerabilities") | .value
            numberOfHotSpots: .__measures[]? | select(.metric == "security_hotspots") | .value
            numberOfDuplications: .__measures[]? | select(.metric == "duplicated_files") | .value
            coverage: .__measures[]? | select(.metric == "coverage") | .value
            mainBranch: .__branch.name
            tags: .tags
```

</details>

### Issue

<details>
<summary>Issue blueprint</summary>

```json showLineNumbers
{
  "identifier": "sonarQubeIssue",
  "title": "SonarQube Issue",
  "icon": "sonarqube",
  "schema": {
    "properties": {
      "type": {
        "type": "string",
        "title": "Type",
        "enum": ["CODE_SMELL", "BUG", "VULNERABILITY"]
      },
      "severity": {
        "type": "string",
        "title": "Severity",
        "enum": ["MAJOR", "INFO", "MINOR", "CRITICAL", "BLOCKER"],
        "enumColors": {
          "MAJOR": "orange",
          "INFO": "green",
          "CRITICAL": "red",
          "BLOCKER": "red",
          "MINOR": "yellow"
        }
      },
      "link": {
        "type": "string",
        "format": "url",
        "icon": "Link",
        "title": "Link"
      },
      "status": {
        "type": "string",
        "title": "Status",
        "enum": ["OPEN", "CLOSED", "RESOLVED", "REOPENED", "CONFIRMED"]
      },
      "assignees": {
        "title": "Assignees",
        "type": "string",
        "icon": "TwoUsers"
      },
      "tags": {
        "type": "array",
        "title": "Tags"
      },
      "createdAt": {
        "type": "string",
        "format": "date-time",
        "title": "Created At"
      }
    }
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {
    "sonarQubeProject": {
      "target": "sonarQubeProject",
      "required": false,
      "title": "SonarQube Project",
      "many": false
    }
  }
}
```

</details>

<details>
<summary>Integration configuration</summary>

```yaml showLineNumbers
createMissingRelatedEntities: true
deleteDependentEntities: true
resources:
  - kind: issues
    selector:
      query: "true"
    port:
      entity:
        mappings:
          blueprint: '"sonarQubeIssue"'
          identifier: .key
          title: .message
          properties:
            type: .type
            severity: .severity
            link: .__link
            status: .status
            assignees: .assignee
            tags: .tags
            createdAt: .creationDate
          relations:
            sonarQubeProject: .project
```

</details>

### Analysis

<details>
<summary>Analysis blueprint</summary>

```json showLineNumbers
{
  "identifier": "sonarQubeAnalysis",
  "title": "SonarQube Analysis",
  "icon": "sonarqube",
  "schema": {
    "properties": {
      "branch": {
        "type": "string",
        "title": "Branch",
        "icon": "GitVersion"
      },
      "fixedIssues": {
        "type": "number",
        "title": "Fixed Issues"
      },
      "newIssues": {
        "type": "number",
        "title": "New Issues"
      },
      "coverage": {
        "title": "Coverage",
        "type": "number"
      },
      "duplications": {
        "type": "number",
        "title": "Duplications"
      },
      "createdAt": {
        "type": "string",
        "format": "date-time",
        "title": "Created At"
      }
    }
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {
    "sonarQubeProject": {
      "target": "sonarQubeProject",
      "required": false,
      "title": "SonarQube Project",
      "many": false
    }
  }
}
```

</details>

<details>
<summary>Integration configuration</summary>

```yaml showLineNumbers
createMissingRelatedEntities: true
deleteDependentEntities: true
resources:
  - kind: analysis
    selector:
      query: "true"
    port:
      entity:
        mappings:
          blueprint: '"sonarQubeAnalysis"'
          identifier: .analysisId
          title: .__commit.message // .analysisId
          properties:
            branch: .__branchName
            fixedIssues: .measures.violations_fixed
            newIssues: .measures.violations_added
            coverage: .measures.coverage_change
            duplications: .measures.duplicated_lines_density_change
            createdAt: .__analysisDate
          relations:
            sonarQubeProject: .__project
```

</details>

## Let's Test It

This section includes a sample response data from SonarQube when a code repository is scanned for quality assurance. In
addition, it includes the entity created from the resync event based on the Ocean configuration provided in the previous
section.

### Payload

Here is an example of the payload structure from SonarQube:

<details>
<summary> Project response data</summary>

```json showLineNumbers
{
  "organization": "peygis",
  "key": "PeyGis_Chatbot_For_Social_Media_Transaction",
  "name": "Chatbot_For_Social_Media_Transaction",
  "isFavorite": true,
  "tags": [],
  "visibility": "public",
  "eligibilityStatus": "COMPLETED",
  "eligible": true,
  "isNew": false,
  "analysisDateAllBranches": "2023-09-09T03:03:20+0200",
  "__measures": [
    {
      "metric": "bugs",
      "value": "6",
      "bestValue": false
    },
    {
      "metric": "code_smells",
      "value": "216",
      "bestValue": false
    },
    {
      "metric": "duplicated_files",
      "value": "2",
      "bestValue": false
    },
    {
      "metric": "vulnerabilities",
      "value": "1",
      "bestValue": false
    },
    {
      "metric": "security_hotspots",
      "value": "8",
      "bestValue": false
    }
  ],
  "__branch": {
    "name": "master",
    "isMain": true,
    "type": "LONG",
    "status": {
      "qualityGateStatus": "ERROR",
      "bugs": 6,
      "vulnerabilities": 1,
      "codeSmells": 216
    },
    "analysisDate": "2023-09-07T14:38:41+0200",
    "commit": {
      "sha": "5b01b6dcb200df0bfd1c66df65be30f9ea5423d8",
      "author": {
        "name": "Username",
        "login": "Username@github",
        "avatar": "9df2ac1caa70b0a67ff0561f7d0363e5"
      },
      "date": "2023-09-07T14:38:36+0200",
      "message": "Merge pull request #21 from PeyGis/test-sonar"
    }
  },
  "__link": "https://sonarcloud.io/project/overview?id=PeyGis_Chatbot_For_Social_Media_Transaction"
}
```

</details>

<details>
<summary> Issue response data</summary>

```json showLineNumbers
{
  "key": "AYhnRlhI0rLhE5EBPGHW",
  "rule": "xml:S1135",
  "severity": "INFO",
  "component": "PeyGis_Chatbot_For_Social_Media_Transaction:node_modules/json-schema/draft-zyp-json-schema-04.xml",
  "project": "PeyGis_Chatbot_For_Social_Media_Transaction",
  "line": 313,
  "hash": "8346d5371c3d1b0d1d57937c7b967090",
  "textRange": {
    "startLine": 313,
    "endLine": 313,
    "startOffset": 3,
    "endOffset": 56
  },
  "flows": [],
  "status": "OPEN",
  "message": "Complete the task associated to this \"TODO\" comment.",
  "effort": "0min",
  "debt": "0min",
  "assignee": "Username@github",
  "author": "email@gmail.com",
  "tags": [],
  "creationDate": "2018-04-06T02:44:46+0200",
  "updateDate": "2023-05-29T13:30:14+0200",
  "type": "CODE_SMELL",
  "organization": "peygis",
  "cleanCodeAttribute": "COMPLETE",
  "cleanCodeAttributeCategory": "INTENTIONAL",
  "impacts": [
    {
      "softwareQuality": "MAINTAINABILITY",
      "severity": "LOW"
    }
  ],
  "__link": "https://sonarcloud.io/project/issues?open=AYhnRlhI0rLhE5EBPGHW&id=PeyGis_Chatbot_For_Social_Media_Transaction"
}
```

</details>

<details>
<summary> Analysis response data</summary>

```json showLineNumbers
{
  "analysisId": "AYpvptJNv89mE9ClYP-q",
  "firstAnalysis": false,
  "measures": {
    "violations_added": "0",
    "violations_fixed": "0",
    "coverage_change": "0.0",
    "duplicated_lines_density_change": "0.0",
    "ncloc_change": "0"
  },
  "branch": {
    "analysisDate": "2023-09-07T12:38:41.279Z",
    "isMain": true,
    "name": "master",
    "commit": {
      "sha": "5b01b6dcb200df0bfd1c66df65be30f9ea5423d8",
      "author": {
        "avatar": "9df2ac1caa70b0a67ff0561f7d0363e5",
        "login": "Username@github",
        "name": "Username"
      },
      "date": "2023-09-07T12:38:36Z",
      "message": "Merge pull request #21 from PeyGis/test-sonar"
    },
    "type": "LONG",
    "status": {
      "qualityGateStatus": "ERROR"
    }
  },
  "__branchName": "master",
  "__analysisDate": "2023-09-07T12:38:41.279Z",
  "__commit": {
    "sha": "5b01b6dcb200df0bfd1c66df65be30f9ea5423d8",
    "author": {
      "avatar": "9df2ac1caa70b0a67ff0561f7d0363e5",
      "login": "Username@github",
      "name": "Username"
    },
    "date": "2023-09-07T12:38:36Z",
    "message": "Merge pull request #21 from PeyGis/test-sonar"
  },
  "__project": "PeyGis_Chatbot_For_Social_Media_Transaction"
}
```

</details>

### Mapping Result

The combination of the sample payload and the Ocean configuration generates the following Port entity:

<details>
<summary> Project entity in Port</summary>

```json showLineNumbers
{
  "identifier": "PeyGis_Chatbot_For_Social_Media_Transaction",
  "title": "Chatbot_For_Social_Media_Transaction",
  "blueprint": "sonarQubeProject",
  "team": [],
  "properties": {
    "organization": "peygis",
    "link": "https://sonarcloud.io/project/overview?id=PeyGis_Chatbot_For_Social_Media_Transaction",
    "lastAnalysisDate": "2023-09-07T12:38:41.000Z",
    "numberOfBugs": 6,
    "numberOfCodeSmells": 216,
    "numberOfVulnerabilities": 1,
    "numberOfHotSpots": 8,
    "numberOfDuplications": 2,
    "mainBranch": "master",
    "tags": []
  },
  "relations": {},
  "icon": "sonarqube"
}
```

</details>

<details>
<summary> Issue entity in Port</summary>

```json showLineNumbers
{
  "identifier": "AYhnRlhI0rLhE5EBPGHW",
  "title": "Complete the task associated to this \"TODO\" comment.",
  "blueprint": "sonarQubeIssue",
  "team": [],
  "properties": {
    "type": "CODE_SMELL",
    "severity": "INFO",
    "link": "https://sonarcloud.io/project/issues?open=AYhnRlhI0rLhE5EBPGHW&id=PeyGis_Chatbot_For_Social_Media_Transaction",
    "status": "OPEN",
    "assignees": "Username@github",
    "tags": [],
    "createdAt": "2018-04-06T00:44:46.000Z"
  },
  "relations": {
    "sonarQubeProject": "PeyGis_Chatbot_For_Social_Media_Transaction"
  },
  "icon": "sonarqube"
}
```

</details>

<details>
<summary> Analysis entity in Port</summary>

```json showLineNumbers
{
  "identifier": "AYpvptJNv89mE9ClYP-q",
  "title": "Merge pull request #21 from PeyGis/test-sonar",
  "blueprint": "sonarQubeAnalysis",
  "team": [],
  "properties": {
    "branch": "master",
    "fixedIssues": 0,
    "newIssues": 0,
    "coverage": 0,
    "duplications": 0,
    "createdAt": "2023-09-07T12:38:41.279Z"
  },
  "relations": {
    "sonarQubeProject": "PeyGis_Chatbot_For_Social_Media_Transaction"
  },
  "icon": "sonarqube"
}
```

</details>
