# orchestration framework


## Service user

```yaml title="config.yaml"
name: inventory-management
version: 1.0.0
services:
  - name: inventory-management-ui
    repo: ui-respository-url
    # We can impose this type to be a specific framework (Say: react).
    # This allows us to have a common frontend that fetches source code from
    # the service repository at build time
    type: ui
  - name: inventory-management-api
    repo: api-respository-url
    # This should mean the service source code has a Dockerfile that exposes
    # a service at $env:PORT and hosts an API at
    # $env:SERVICE_BASE_URL/**
    # and has documentation at
    # $env:SERVICE_BASE_URL/docs
    type: api
  - name: inventory-management-barcode-sensor
    repo: sensor-respository-url
    # This means that this is a special type of service that is used to 
    # collect data from specific k8s nodes
    # It should have a Dockerfile (can access host machine) that uploads
    # collected data to data storage
    type: agent
    tagname: barcode-sensor-node
  - name: inventory-management-db
    repo: db-respository-url
    # This means the folder has a Dockerfile that exposes a port
    type: db
```

**Action**: tell the orchestrator to track this config (git, kubernetes, mdns)

## Orchestrator

**Respond to Action**: new config detected run `logic.sh`


``` mermaid
sequenceDiagram
    Component->Orchestrator: {name}
    Component->Orchestrator: {version}
    loop services
        Component-->Orchestrator: {service.name}
        Component-->Orchestrator: {service.repo}
        Component-->UI_BUILD: {service.type}
        Component-->API_BUILD: {service.type}
        Component-->AGENT_BUILD: {service.type}
    end
```

```bash title="logic.sh"
Get source code

Create a k8s namespace (given: {name}, {version})

for each service in {services}

if service is 'ui'
add {service.repo} to the common frontend build

if the service is 'api'
build the {service.repo}/Dockerfile with env:
  PORT: 8080
  SERVICE_BASE_URL: {name}/{service.name}
  this is for the frontend to have a common url for api access
    eg: /inventory-management/inventory-management-api/**
    docs: /inventory-management/inventory-management-api/docs
  frontend can assume other services exist too
  But must fail gracefully
    eg: /stock/provider/**
    docs: /stock/provider/docs

if the service is 'agent'
build the {service.repo}/Dockerfile
schedule the deployment to run on {service.tagname} node
*this means it now has access to service apis through k8s*
*and host machine capabilities*
```