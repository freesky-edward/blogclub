# PostgreSQL RDS Service Broker Introduction

This is a [Cloud Foundry Service Broker](https://docs.cloudfoundry.org/services/overview.html) for [Huawei Relational Database Service (RDS)](http://www.huaweicloud.com/en-us/product/rds.html) supporting [PostgreSQL](http://support.huaweicloud.com/en-us/usermanual-rds/en-us_topic_0044262672.html) RDS Databases.

## Installation

### Locally

Using the standard `go install` (you must have [Go](https://golang.org/) already installed in your local machine):

```
$ go install github.com/chenyingkof/rds-broker
$ rds-broker -port=3000 -config=<path-to-your-config-file>
```

### Cloud Foundry

The broker can be deployed to an already existing [Cloud Foundry](https://www.cloudfoundry.org/) installation:

```
$ git clone https://github.com/chenyingkof/rds-broker.git
$ cd rds-broker
```

Modify the [configuration file](https://github.com/chenyingkof/rds-broker/blob/master/config-sample.json) to include your RDS authentication configurations and some parameters or configurations for providing the DB Instances in the [sample configuration file](https://github.com/chenyingkof/rds-broker/blob/master/config-sample.json). Then you can push the broker to your [Cloud Foundry](https://www.cloudfoundry.org/) environment:

```
$ cp config-sample.json config.json
$ cf push rds-broker
```

## Usage

### Managing Service Broker

Configure and deploy the broker. Then:

1. Check that your Cloud Foundry installation supports [Service Broker API](https://docs.cloudfoundry.org/services/api.html)
2. [Register the broker](https://docs.cloudfoundry.org/services/managing-service-brokers.html#register-broker) within your Cloud Foundry installation;
3. [Make Services and Plans public](https://docs.cloudfoundry.org/services/access-control.html#enable-access);
4. Depending on your Cloud Foundry settings, you might also need to create/bind an [Application Security Group](https://docs.cloudfoundry.org/adminguide/app-sec-groups.html) to allow access to the RDS DB Instances.

### Integrating Service Instances with Applications

Application Developers can start to consume the services using the standard [CF CLI commands](https://docs.cloudfoundry.org/devguide/services/managing-services.html).

Depending on the [broker configuration](https://github.com/chenyingkof/rds-broker/blob/master/CONFIGURATION.md#rds-broker-configuration), Application Developers can use the Credentials information from the
response of broker Bind call for accessing DB Instances from RDS.


## Contributing

In the spirit of [free software](http://www.fsf.org/licensing/essays/free-sw.html), **everyone** is encouraged to help improve this project.

Here are some ways *you* can contribute:

* by using prerelease versions or master branch.
* by reporting bugs
* by suggesting new features
* by writing or editing documentation
* by writing specifications
* by writing code (**no patch is too small**: fix typos, add comments, clean up inconsistent whitespace)
* by refactoring code
* by closing [issues](https://github.com/chenyingkof/rds-broker/issues)
* by reviewing patches

### Submitting an Issue

We use the [GitHub issue tracker](https://github.com/chenyingkof/rds-broker/issues) to track bugs and features. Before submitting a bug report or feature request, check to make sure it hasn't already been submitted. You can indicate support for an existing issue by voting it up. When submitting a bug report, please include a [Gist](http://gist.github.com/) that includes a stack trace and any details that may be necessary to reproduce the bug, including your Golang version and operating system. Ideally, a bug report should include a pull request with failing specs.

### Submitting a Pull Request

1. Fork the project.
2. Create a topic branch.
3. Implement your feature or bug fix.
4. Commit and push your changes.
5. Submit a pull request.

# Configuration

A sample configuration can be found at [config-sample.json](https://github.com/chenyingkof/rds-broker/blob/master/config-sample.json).

## General Configuration

| Option     | Required | Type   | Description
|:-----------|:--------:|:------ |:-----------
| log_level  | Y        | String | Broker Log Level (DEBUG, INFO, ERROR, FATAL)
| username   | Y        | String | Broker Auth Username
| password   | Y        | String | Broker Auth Password
| rds_config | Y        | Hash   | [RDS Broker configuration](https://github.com/chenyingkof/rds-broker/blob/master/CONFIGURATION.md#rds-broker-configuration)

## RDS Broker Configuration

| Option                         | Required | Type    | Description
|:-------------------------------|:--------:|:------- |:-----------
| identity_endpoint              | Y        | String  | The identity endpoint URL of Keystone
| ca                             | Y        | String  | Keystone Auth ca file
| username                       | Y        | String  | Keystone Auth username
| password                       | Y        | String  | Keystone Auth password
| domain_name                    | Y        | String  | Keystone Auth domain name
| project_name                   | Y        | String  | Keystone Auth project name
| project_id                     | Y        | String  | Keystone Auth project id
| region                         | Y        | String  | Keystone Auth region
| db_prefix                      | Y        | String  | Prefix to add to RDS DB Identifiers
| catalog                        | Y        | Hash    | [RDS Broker catalog](https://github.com/chenyingkof/rds-broker/blob/master/CONFIGURATION.md#rds-broker-catalog)

1 If the broker api will be deployed in the cloudfoundry, the value of ca could be the absolute path of this project. For example: ca =  ./ca.crt. The ca.crt can be deployed
in the rds-broker itself.
2 If the ca file is not needed for Keystone authentication, the value of ca can be set empty string.


## RDS Broker catalog

Please refer to the [Catalog Documentation](https://docs.cloudfoundry.org/services/api.html#catalog-mgmt) for more details about these properties.

### Catalog

| Option   | Required | Type      | Description
|:---------|:--------:|:--------- |:-----------
| services | N        | []Service | A list of [Services](https://github.com/chenyingkof/rds-broker/blob/master/CONFIGURATION.md#service)

### Service

| Option                        | Required | Type          | Description
|:------------------------------|:--------:|:------------- |:-----------
| id                            | Y        | String        | An identifier used to correlate this service in future requests to the catalog
| name                          | Y        | String        | The CLI-friendly name of the service that will appear in the catalog. All lowercase, no spaces
| description                   | Y        | String        | A short description of the service that will appear in the catalog
| bindable                      | N        | Boolean       | Whether the service can be bound to applications
| tags                          | N        | []String      | A list of service tags
| metadata.displayName          | N        | String        | The name of the service to be displayed in graphical clients
| metadata.imageUrl             | N        | String        | The URL to an image
| metadata.longDescription      | N        | String        | Long description
| metadata.providerDisplayName  | N        | String        | The name of the upstream entity providing the actual service
| metadata.documentationUrl     | N        | String        | Link to documentation page for service
| metadata.supportUrl           | N        | String        | Link to support for the service
| requires                      | N        | []String      | A list of permissions that the user would have to give the service, if they provision it (only `syslog_drain` is supported)
| plan_updateable               | N        | Boolean       | Whether the service supports upgrade/downgrade for some plans
| plans                         | N        | []ServicePlan | A list of [Plans](https://github.com/chenyingkof/rds-broker/blob/master/CONFIGURATION.md#service-plan) for this service
| dashboard_client.id           | N        | String        | The id of the Oauth2 client that the service intends to use
| dashboard_client.secret       | N        | String        | A secret for the dashboard client
| dashboard_client.redirect_uri | N        | String        | A domain for the service dashboard that will be whitelisted by the UAA to enable SSO

### Service Plan

| Option               | Required | Type          | Description
|:---------------------|:--------:|:------------- |:-----------
| id                   | Y        | String        | An identifier used to correlate this plan in future requests to the catalog
| name                 | Y        | String        | The CLI-friendly name of the plan that will appear in the catalog. All lowercase, no spaces
| description          | Y        | String        | A short description of the plan that will appear in the catalog
| metadata.bullets     | N        | []String      | Features of this plan, to be displayed in a bulleted-list
| metadata.costs       | N        | Cost Object   | An array-of-objects that describes the costs of a service, in what currency, and the unit of measure
| metadata.displayName | N        | String        | Name of the plan to be display in graphical clients
| free                 | N        | Boolean       | This field allows the plan to be limited by the non_basic_services_allowed field in a Cloud Foundry Quota
| rds_properties       | Y        | RDSProperties | [RDS Properties](https://github.com/chenyingkof/rds-broker/blob/master/CONFIGURATION.md#rds-properties)

## RDS Properties

Please refer to the [Huawei Relational Database Service Documentation](http://support.huaweicloud.com/en-us/rds_gls/index.html) for more details about these properties.

| Option                          | Required | Type      | Description
|:--------------------------------|:--------:|:--------- |:-----------
| datastore_type                  | Y        | String    | Specifies the DB engine. Currently, MySQL, PostgreSQL, and Microsoft SQL Server are supported. The value is PostgreSQL.
| datastore_version               | Y        | String    | Specifies the DB instance version.
| flavor_name                     | N        | String    | Specifies the specification ID compliant with the UUID format.
| flavor_id                       | Y        | String    | Specifies the specification name.
| volume_type                     | Y        | String    | Specifies the volume type. Valid value: It must be COMMON (SATA) or ULTRAHIGH (SSD) and is case-sensitive.
| volume_size                     | Y        | Integer   | Specifies the volume size. Its value must be a multiple of 10 and the value range is 100 GB to 2000 GB.
| region                          | Y        | String    | Specifies the region ID. Valid value: The value cannot be empty. For details about how to obtain this parameter value, see Regions and Endpoints.
| availability_zone               | Y        | String    | Specifies the ID of the AZ. Valid value: The value cannot be empty. For details about how to obtain this parameter value, see Regions and Endpoints.
| vpc_id                          | Y        | String    | Specifies the VPC ID.
| subnet_id                       | Y        | String    | Specifies the UUID for nics information.
| security_group_id               | Y        | String    | Specifies the security group ID which the RDS DB instance belongs to.
| db_port                         | Y        | String    | Specifies the database port number.
| backup_strategy_starttime       | Y        | String    | Indicates the backup start time that has been set. The backup task will be triggered within one hour after the backup start time.
| backup_strategy_keepdays        | Y        | Integer   | Specifies the number of days to retain the generated backup files. Its value range is 0 to 35.
| db_password                     | Y        | String    | Specifies the password for user root of the database. (Valid value: The value cannot be empty and should contain 8 to 32 characters, including uppercase and lowercase letters, digits, and the following special characters: ~!@#%^*-_=+?)
| db_username                     | Y        | String    | Specifies the username for user root of the database. The default value of username is root.
| db_name                         | Y        | String    | Specifies the name for user root of the database. The default value of database name is postgres.


## RDS Broker NOTE:

This rds broker will provide the database instance from Relational Database Service to the applications in Cloud Foundry.

The Services action of this rds broker API will return a service catalog about Relational Database Service with different specifications in plan.

The Provision action of this rds broker API will create a database instance in Relational Database Service.

The Deprovision action of this rds broker API will delete the created database instance in Relational Database Service.

The LastOperation action of this rds broker API will return the status of the created database instance in Relational Database Service.

The Bind action of this rds broker API will return the credentials information about the created database instance in Relational Database Service.
These credentials information can be used by the applications in CloudFoundry.

The Unbind action of this rds broker API will return unbind the credentials information about the created database instance in Relational Database Service.

The Update action of this rds broker API will update some information for Relational Database Service database instance.
