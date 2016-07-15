# Build Helpers

This folder contains several helper scripts for automatically building deployable, versioned artifacts of your apps:

* `build-docker-image`: This script can be used to automatically build a Docker image. It runs a Docker build once
  with the `latest` tag and once with the sha1 of the most recent commit, runs an optional test command against the
  images, pushes the new images to a Docker registry, and writes the sha1 tag to a properties file. This script is
  meant to be run from a CI job and the properties file is a convenient way to pass information about the new Docker
  image to another CI job.
* `build-packer-artifact`: This script can be used to automatically build an artifact, such as an AMI, defined in a
  Packer template. It runs a Packer build, runs an optional test command to verify the new artifact, extracts the
  artifact ID from the build, and writes the ID to a properties file. This script is meant to be run from a CI job and
  the properties file is a convenient way to pass information about the new artifact to another CI job.

## Using the helper scripts in your code

You can install these scripts using the [Gruntwork Installer](https://github.com/gruntwork-io/gruntwork-installer):

```bash
gruntwork-install --module-name "build-helpers" --repo "https://github.com/gruntwork-io/module-ci" --tag "0.0.1"
```

## Properties files

After a successful build, the build helper scripts write the versions of the artifacts they produce to a properties
file. The idea is that this properties file is easy to use in other scripts or other CI jobs to deploy those artifacts.

The `build-docker-image` script adds an entry to the properties file of the form:

```
DOCKER_IMAGE_TAG=<IMAGE_TAG>
```

The `build-packer-artifact` script adds an entry to the properties file of the form:

```
ARTIFACT_ID=<ARTIFACT_ID>
```

## Examples

The examples below should give you an idea of how to use the scripts. Run the scripts above with the `--help` flag to
see full documentation.

Imagine you have a Packer template `templates/build.json` that specifies how to build an AMI for your app. You could
set up automatic deployment for this app using two Jenkins CI jobs: `build-app` and `deploy-app`.

The `build-app` CI job would first use the `build-packer-artifact` script to automatically build your AMI:

```bash
build-packer-artifact --packer-template-path templates/build.json --output-properties-file artifacts.properties
```

Next, the `build-app` CI job would then pass the contents of `artifacts.properties` directly to the `deploy-app` CI
job using the [Jenkins Parameterized Trigger
Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Parameterized+Trigger+Plugin). The `deploy-app` CI job, in turn,
would be a [parameterized build](https://wiki.jenkins-ci.org/display/JENKINS/Parameterized+Build) that takes as input
a parameter called `ARTIFACT_ID` (the same parameter name that's in the `artifacts.properties` file) and use it, along
with the scripts in the [terraform-helpers module](/modules/terraform-helpers) to automatically deploy the new AMI:

```bash
cd templates
terraform-update-variable --name rails_app_version --value $ARTIFACT_ID
terraform-deploy --remote-state-bucket my_s3_bucket --aws-region us-east-1
```