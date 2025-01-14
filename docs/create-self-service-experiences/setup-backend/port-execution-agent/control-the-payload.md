# Control the payload

Some of the 3rd party applications that you may want to integrate with may not accept the raw payload incoming from
Port's
self-service actions. The Port agent allows you to control the payload that is sent to every 3rd party application.

You can alter the requests sent to your third-party application by providing a payload mapping config file when
deploying the
Port-agent container.

### Control the payload mapping

The payload mapping file is a JSON file that specifies how to transform the request sent to the Port agent to the
request that is sent to the third-party application.

The payload mapping file is mounted to the Port agent as a volume. The path to the payload mapping file is set in the
`CONTROL_THE_PAYLOAD_CONFIG_PATH` environment variable. By default, the Port agent will look for the payload mapping
file at `~/control_the_payload_config.json`.

The payload mapping file is a json file that contains a list of mappings. Each mapping contains the request fields that
will be overridden and sent to the 3rd party application.

You can see examples showing how to deploy the Port agent with different mapping configurations for various common use
cases below.

Each of the mapping fields can be constructed by JQ expressions. The JQ expression will be evaluated against the
original payload that is sent to the port agent from Port and the result will be sent to the 3rd party application.

Here is the mapping file schema:

```showLineNumbers
[ # Can have multiple mappings. Will use the first one it will find with enabled = True (Allows you to apply mapping over multiple actions at once)
  {
      "enabled": bool || JQ,
      "url": JQ, # Optional. default is the incoming url from port
      "method": JQ, # Optional. default is POST. Should return one of the following string values POST / PUT / DELETE / GET
      "headers": dict[str, JQ], # Optional. default is {}
      "body": ".body", # Optional. default is the whole payload incoming from Port.
      "query": dict[str, JQ] # Optional. default is {},
      "report" { # Optional. Used to report the run status back to Port right after the request is sent to the 3rd party application
        "status": JQ, # Optional. Should return the wanted runs status
        "link": JQ, # Optional. Should return the wanted link or a list of links
        "summary": JQ, # Optional. Should return the wanted summary
        "externalRunId": JQ # Optional. Should return the wanted external run id
      }
  }
]
```

**The body can be partially constructed by json as follows:**

```json showLineNumbers
{
  "body": {
    "key": 2,
    "key2": {
      "key3": ".im.a.jq.expression",
      "key4": "\"im a string\""
    }
  }
}
```

### The incoming message to base your mapping on

```json showLineNumbers
{
  "action": "action_identifier",
  "resourceType": "run",
  "status": "TRIGGERED",
  "trigger": {
    "by": {
      "orgId": "org_XXX",
      "userId": "auth0|XXXXX",
      "user": {
        "email": "executor@mail.com",
        "firstName": "user",
        "lastName": "userLastName",
        "phoneNumber": "0909090909090909",
        "picture": "https://s.gravatar.com/avatar/dd1cf547c8b950ce6966c050234ac997?s=480&r=pg&d=https%3A%2F%2Fcdn.auth0.com%2Favatars%2Fga.png",
        "providers": ["port"],
        "status": "ACTIVE",
        "id": "auth0|XXXXX",
        "createdAt": "2022-12-08T16:34:20.735Z",
        "updatedAt": "2023-11-13T15:11:38.243Z"
      }
    },
    "origin": "UI",
    "at": "2023-11-13T15:20:16.641Z"
  },
  "context": {
    "entity": "e_iQfaF14FJln6GVBn",
    "blueprint": "kubecostCloudAllocation",
    "runId": "r_HardNzG6kzc9vWOQ"
  },
  "payload": {
    "entity": {
      "identifier": "e_iQfaF14FJln6GVBn",
      "title": "myEntity",
      "icon": "Port",
      "blueprint": "myBlueprint",
      "team": [],
      "properties": {},
      "relations": {},
      "createdAt": "2023-11-13T15:24:46.880Z",
      "createdBy": "auth0|XXXXX",
      "updatedAt": "2023-11-13T15:24:46.880Z",
      "updatedBy": "auth0|XXXXX"
    },
    "action": {
      "invocationMethod": {
        "type": "WEBHOOK",
        "agent": true,
        "synchronized": false,
        "method": "POST",
        "url": "https://myGitlabHost.com"
      },
      "trigger": "DAY-2"
    },
    "properties": {},
    "censoredProperties": []
  }
}
```

## Report action status back to Port

The agent is capable of reporting the action status back to Port using the `report` field in the mapping.

The report request will be sent to the Port API right after the request to the 3rd party application is sent and update
the run status in Port.

The agent uses the JQ in the `report` field to construct the report request body.

The available fields are:

- `status` - The status of the run. Can be one of the following values: `SUCCESS` / `FAILURE`
- `link` - A link to the run in the 3rd party application. Can be a string or a list of strings.
- `summary` - A string summary of the run.
- `externalRunId` - The external run id in the 3rd party application. The external run id is used to allow a search of
  the action runs in Port by the external run id.

The report mapping can use the following fields:

`.body` - The incoming message as mentioned [above](#the-incoming-message-to-base-your-mapping-on)
`.request` - The request that was calculated using the control the payload mapping and sent to the 3rd party application
`.response` - The response that was received from the 3rd party application

The `response` field contains the following fields:

- `statusCode` - The status code of the response
- `json` - The response body as a json object
- `text` - The response body as a string
- `headers` - The response headers as a json object
