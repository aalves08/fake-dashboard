# UI Automation execution on Jenkins

## Infrastructure Configuration

The configuration is handled by the [init.sh](./init.sh) script in this repository.

Basically what it does is a setup of a Linux node at a user-level of the software needed to execute the execution script located in the Dashboard tests [Corral Package](https://github.com/rancherlabs/corral-packages).

It configures a `WORKSPACE` folder in the *PATH* to add the necessary binaries.

In Jenkins executors `WORKSPACE` is a predefined variable to a temporary folder in the *jenkins* user home.

- Golang - To build the corral package images.
- Corral - For executng the corral package to build the remote Node for the tests
- yq - For updating the corral package configuration values

If the `JOB_TYPE` is `recurring` that will make use of the `rancher` package to spin up a Rancher server for the dashboards tests to run on it.

## Corral configuration

The initialization script makes use of the `corral config vars` option to set the dashboard tests corral package values.

The goal is to have logic in the script for configuring both a Rancher instance and the Dashboard tests node that runs:

 `Dashboard tests node -> Rancher setup`

## Run locally - needs remote aws provider

It is possible to run this locally or in a remove Linux instance.

There's however required environment variables set for configuring the script, download the binary dependencies and execute corral.

This design is mainly due to how the Jenkins Jobs manage the configuration using environment variables.

Some environment have default values thus making them optional others like the following are required:

- `WORKSPACE`
  - This is defined in Jenkins but it isn't a system variable on Linux.
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `AWS_ZONE_ID`
- `AWS_DOMAIN`
- `AWS_AMI`
- `AWS_SECURITY_GROUP`
- `AWS_SUBNET`
- `AWS_VPC`
- `AWS_VOLUME_TYPE`
- `AWS_VOLUME_IOPS`
- `AWS_SSH_USER`
- `AWS_INSTANCE_TYPE`
- `RANCHER_TYPE`
  - This var defines the logic of how to execute the tests by the corral package.
- `CYPRESS_TAGS`

*For more variables or variables updates take a look at the [init.sh](./init.sh) script.*

Folder structure:

- `WORKSPACE`
  - `corral-packages` - cloned repo folder.
  - `dashboard` - cloned repo folder
  - `bin` - folder with binaries and set in the `PATH`

From `WORKSPACE`:

`dashboard/cypress/jenkins/init.sh`

The expected and default Dashboard corral package image path is:

`${WORKSPACE}/corral-packages/dist/aws-dashboard-tests-t3a.xlarge`

That is generated by `make` during the `init.sh` execution.

### Use run.sh from the corral package directly

The main `run.sh` script has all the logic to directly run the tests without creating the remote AWS node. However that's not fully tested and some modifications might be required to make it work.

## CORRAL PACKAGE

The Dashboard Tests corral package basically create an `AWS` ephimeral node to execute the UI tests on a `cypress` docker container.

The Docker image used is the latest `factory` from the `cypress-io/cypress-docker-images` repository [folder](https://github.com/cypress-io/cypress-docker-images/tree/master/factory).
That image can recieve arguments for `yarn`, `node`, `cypress` and browsers. This allows the Jenkins job to set these and run.

A [run.sh](https://github.com/izaac/corral-packages/blob/dashboard_tests_recurring/templates/dashboard-tests/overlay/tmp/run.sh) script is executed on the remote node during the node creation.

The script has logic for three different environment configurations - `rancher_type`:

- Execute the tests on an `existing` Rancher setup.
- Execute the tests in a Rancher instance running along the tests in the `local` node.
- Do `recurring` execution that on an `existing` ephimeral Rancher created by the automation for Jenkins.

The corral package takes the configuration from the updated/edited YAML files for the `recurring` type and all the variables set by the `init.sh` script with `corral config vars`.

For non recurring types it just make use of the `corral config vars`, as the YAML files are only used for the ephimeral automation Rancher node.

The workflow in `run.sh` of the different `rancher_type`(s).

### Existing

- Install all node modules needed for cypress reporting on the `dashboard` cloned repo.
- Utilize an existing `rancher_host` username and password.
- With the username and password, execute a Docker instance with the UI tests
- Generate jUnit and html reports.

### Recurring

- Install all node modules needed for cypress reporting on the `dashboard` cloned repo.
- Grab the information of the ephemeral Rancher automation instance from `corral config vars`
- With that information like the Rancher username and password, execute a Docker instance with the UI tests
- Generate jUnit and html reports.

### Local

Similar to Drone without the remote reporting and this runs Rancher Docker container and the cypress container.
This is less used in Jenkins and might be a good option for local testing.

- Install all node modules needed for cypress reporting on the `dashboard` cloned repo.
- Build UI assets and setup a Docker Rancher instance with them. (Similar to Drone).
- Attach a second Docker container to the Rancher instance network and execute the UI tests in it.
- Generate jUnit and html reports.
