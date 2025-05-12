# Deploying a Flask App with Failure Flags Sidecar on Cloud Foundry

## Step 1: Download and Extract the Failure Flags Sidecar

Choose the appropriate architecture (**amd64** or **arm64**) and run the command below in your project's root directory:

```bash
# For x86_64 (amd64)
mkdir -p bin
curl -sSL https://assets.gremlin.com/packages/failure-flags-sidecar/latest/x86_64/failure-flags-sidecar-linux.tar.gz \
  | tar xz --strip-components=2 -C ./bin ./bin/failure-flags-sidecar-amd64-linux
```

```
# For ARM64
mkdir -p bin
curl -sSL https://assets.gremlin.com/packages/failure-flags-sidecar/latest/arm64/failure-flags-sidecar-linux.tar.gz \
  | tar xz --strip-components=2 -C ./bin ./bin/failure-flags-sidecar-arm64-linux
```

Ensure your sidecar executable is in the `bin/` directory:

```
bin/failure-flags-sidecar-amd64-linux
# OR
bin/failure-flags-sidecar-arm64-linux
```

## Step 2: Create `vars.yml`

Your `manifest.yml` references secrets using `((placeholder))` syntax. Create a local `vars.yml` file (do not commit to Git):

```yaml
aws_access_key_id: "YOUR_ACCESS_KEY_ID"
aws_secret_access_key: "YOUR_SECRET_ACCESS_KEY"

gremlin_team_id: "YOUR_GREMLIN_TEAM_ID"
gremlin_team_certificate: |-
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
gremlin_team_private_key: |-
  -----BEGIN EC PRIVATE KEY-----
  ...
  -----END EC PRIVATE KEY-----
```

* Replace placeholders with actual credentials.
* Add `vars.yml` to your `.gitignore` file to prevent accidental commits.

## Step 3: Configure `manifest.yml`

Here's an example Cloud Foundry manifest for your Flask application with a Gremlin sidecar:

```yaml
---
applications:
  - name: s3-failure-flags-app
    memory: 512M
    disk_quota: 1G
    instances: 1
    buildpacks:
      - python_buildpack
    path: .
    env:
      S3_BUCKET: "commoncrawl"
      DEBUG_MODE: "false"
      CLOUD: "pcf"
      AWS_ACCESS_KEY_ID: ((aws_access_key_id))
      AWS_SECRET_ACCESS_KEY: ((aws_secret_access_key))
      FAILURE_FLAGS_ENABLED: "true"
      GREMLIN_SIDECAR_ENABLED: "true"
      GREMLIN_TEAM_ID: ((gremlin_team_id))
      GREMLIN_TEAM_CERTIFICATE: ((gremlin_team_certificate))
      GREMLIN_TEAM_PRIVATE_KEY: ((gremlin_team_private_key))
      GREMLIN_DEBUG: "true"
      SERVICE_NAME: "s3-failure-flags-app"
    sidecars:
      - name: gremlin-sidecar
        process_types: ["web"]
        memory: 256M
        disk_quota: 256M
        command: "./bin/failure-flags-sidecar-amd64-linux" # or arm64 binary
```

Adjust paths, environment variables, and resource limits as necessary.

## Step 4: Push Application to Cloud Foundry

From your project directory, deploy the application:

```bash
cf push --vars-file vars.yml
```

## Step 5: Validate the Cloud Foundry App

Check the status of your deployed application:

```bash
cf apps
```

To see detailed health and status information:

```bash
cf app s3-failure-flags-app
```

Ensure that your application status is "running" and that the number of instances matches your manifest configuration.

## Step 6: Inspect Application Logs

### Tail Logs Continuously

Tail logs to monitor application runtime behavior in real-time:

```bash
cf logs s3-failure-flags-app
```

### View Recent Logs

Retrieve recent logs for quick inspection:

```bash
cf logs s3-failure-flags-app --recent
```

Check logs regularly to confirm your app and Gremlin sidecar are working correctly.

## References

* [Deploying Failure Flags on Pivotal Cloud Foundry (PCF)](https://www.gremlin.com/docs/deploying-failure-flags-on-pivotal-cloud-foundry-pcf)
* [Cloud Foundry Sidecar Processes](https://docs.cloudfoundry.org/devguide/sidecars.html)
* [Manifest Sidecar Attribute Reference](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#sidecars)

