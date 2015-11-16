---
title: Deployments
weight: 40
menu:
  main:
    parent: api-reference
---

# Deployments

Deployments are non-static entities in the Vamp eco-system. They represent runtime structures so any changes to them will take time to execute and can possibly fail. Most API calls to the `/deployments` endpoint will therefore return a `202: Accepted` return code, indicating the asynchronous nature of the call.

Deployments have a set of sub resources: **SLA's**, **scales** and **routings**. These are instantiations of their static counterparts.

Please check the notes on using [pagination](/documentation/api-reference/#pagination) and [json and yaml content types](/documentation/api-reference/#content-types) on how to effectively use the REST api.

## List deployments

	GET /api/v1/deployments

| parameter         | options           | default          | description      |
| ----------------- |:-----------------:|:----------------:| ----------------:|
| as_blueprint      | true or false     | false            | exports each deployment as a valid blueprint. This can be used together with the header `Accept: application/x-yaml` to export in YAML format instead of the default JSON. |
| expand_references | true or false     | false            | all breed references will be replaced (recursively) with full breed definitions. It will be applied only if `?as_blueprint=true`.
| only_references   | true or false     | false            | all breeds will be replaced with their references. It will be applied only if `?as_blueprint=true`.

## Get a single deployment

Lists all details for one specific deployment.

    GET /api/v1/deployments/:id

| parameter         | options           | default          | description      |
| ----------------- |:-----------------:|:----------------:| ----------------:|
| as_blueprint      | true or false     | false            | exports the deployment as a valid blueprint. This can be used together with the header `Accept: application/x-yaml` to export in YAML format instead of the default JSON. |
| expand_references | true or false     | false            | all breed references will be replaced (recursively) with full breed definitions. It will be applied only if `?as_blueprint=true`.
| only_references   | true or false     | false            | all breeds will be replaced with their references. It will be applied only if `?as_blueprint=true`.

## Create deployment using a blueprint

Creates a new deployment

	POST /api/v1/deployments

| parameter     | options           | default          | description      |
| ------------- |:-----------------:|:----------------:| ----------------:|
| validate_only | true or false     | false            | validates the blueprint and returns a `202 Accepted` if the blueprint is valid for deployment. This can be used together with the header `Accept: application/x-yaml` to return the result in YAML format instead of the default JSON.     

## Update a deployment using a blueprint

Updates the settings of a specific deployment.

    PUT /api/v1/deployments/:id

| parameter     | options           | default          | description      |
| ------------- |:-----------------:|:----------------:| ----------------:|
| validate_only | true or false     | false            | validates the blueprint and returns a `202 Accepted` if the deployment after the update would be still valid. This can be used together with the header `Accept: application/x-yaml` to return the result in YAML format instead of the default JSON. 

## Delete a deployment using a blueprint

Deletes all or parts of a deployment.        

    DELETE /api/v1/deployments/:id

| parameter     | options           | default          | description      |
| ------------- |:-----------------:|:----------------:| ----------------:|
| validate_only | true or false     | false            | validates the blueprint and returns a `202 Accepted` if the deployment after the (partial) deletion would be still valid. Actual delete is not performed.

In contrast to most API's, doing a `DELETE` in Vamp takes a request body that designates what part of the deployment should be deleted. This allows you to remove specific services, clusters of the whole deployment.

> Note: A `DELETE` on a deployment with an empty request body will not delete anything.

The most common way to specify what you want to delete is by exporting the target deployment as a blueprint using the `?as_blueprint=true` parameter. You then either programmatically or by hand edit the resulting blueprint and specify which of the services you want to delete. You can also use the blueprint as a whole in the `DELETE` request. The result is the removal of the full deployment. 

**example:**

This is our (abbreviated) deployment in YAML format. We have two clusters. The first cluster 'frontend' has two services.
We have left out some keys like `scale`, `routing` and `servers` among others as they have no effect on this specific use case.

		GET /api/v1/deployment/3df5c37c-5137-4d2c-b1e1-1cb3d03ffcd?as_blueprint=true

```yaml
name: 3df5c37c-5137-4d2c-b1e1-1cb3d03ffcdd
endpoints:
  frontend.port: '9050'
clusters:
  frontend:
    services:
    - breed:
        name: monarch_front:0.1
        deployable: magneticio/monarch:0.1
        ports:
          port: 8080/http
        constants: {}
        dependencies:
          backend:
            ref: monarch_backend:0.3
    - breed:
        name: monarch_front:0.2
        deployable: magneticio/monarch:0.2
        ports:
          port: 8080/http
        dependencies:
          backend:
            ref: monarch_backend:0.3
  backend:
    services:
    - breed:
        name: monarch_backend:0.3
        deployable: magneticio/monarch:0.3
        ports:
          jdbc: 8080/http
        environment_variables: {}
```    

If we want to delete the first service in the `frontend` cluster, we use the following blueprint as the request body in the `DELETE` action.

	DELETE /api/v1/deployments/3df5c37c-5137-4d2c-b1e1-1cb3d03ffcdd
		
```yaml
name: 3df5c37c-5137-4d2c-b1e1-1cb3d03ffcdd
clusters:
  frontend:
    services:
    - breed:
        ref: monarch_front:0.1
```        		

If we want to delete the whole deployment, we just specify all the clusters and services.

	DELETE /api/v1/deployments/3df5c37c-5137-4d2c-b1e1-1cb3d03ffcdd
		
```yaml
name: 3df5c37c-5137-4d2c-b1e1-1cb3d03ffcdd
clusters:
  frontend:
    services:
    - breed:
        ref: monarch_front:0.1
    - breed:
        ref: monarch_front:0.2
  backend:
    services:
    - breed:
        ref: monarch_backend:0.3
```        		    

# Deployment SLA's

## Get a deployment SLA

Lists all details for a specific SLA that's part of a specific cluster.

	GET /api/v1/deployments/:id/clusters/:name/sla
	
## Set a deployment SLA

Creates or updates a specific deployment SLA.

	POST|PUT /api/v1/deployments/:id/clusters/:name/sla
	
## Delete a deployment SLA

Deletes as specific deployment SLA.

	DELETE /api/v1/deployments/:id/clusters/:name/sla


# Deployment scales

Deployment scales are singular resources: you only have one scale per service. Deleting a scale is not a meaningfull action.

## Get a deployment scale

Lists all details for a specific deployment scale that's part of a service inside a cluster.

	GET /api/v1/deployments/:id/clusters/:name/services/:name/scale
	
## Set a deployment scale	

Updates a deployment scale.

	POST|PUT /api/v1/deployments/:id/clusters/:name/services/:name/scale

# Deployment routings

Deployment routing are singular resources: you only have one routing per service. Deleting a routing is not a meaningful action.

## Get a deployment routing

Lists all details for a specific deployment routing that's part of a service inside a cluster.

	GET /api/v1/deployments/:id/clusters/:name/services/:name/routing
	
## Set a deployment routing	

Updates a deployment routing.

	POST|PUT /api/v1/deployments/:id/clusters/:name/services/:name/routing