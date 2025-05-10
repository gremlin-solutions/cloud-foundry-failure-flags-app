# Deploying a Flask App and the Gremlin Failure Flags Sidecar on Cloud Foundry

## 1. Download and Extract the Failure Flags Sidecar

Download and extract the Failure Flags sidecar for your CPU architecture. Choose **amd64** (x86_64) or **arm64**:

```bash
# ── For x86_64 (amd64) ─────────────────────────────────────────────────────────
curl -sSL https://assets.gremlin.com/packages/failure-flags-sidecar/latest/x86_64/failure-flags-sidecar-linux.tar.gz \
  | tar xz \
      --strip-components=2 \
      -C . \
      ./bin/failure-flags-sidecar-amd64-linux

# ── For ARM64 ─────────────────────────────────────────────────────────────────
curl -sSL https://assets.gremlin.com/packages/failure-flags-sidecar/latest/arm64/failure-flags-sidecar-linux.tar.gz \
  | tar xz \
      --strip-components=2 \
      -C . \
      ./bin/failure-flags-sidecar-arm64-linux
```

## 2. Create `vars.yml`

The `manifest.yml` uses `(( placeholder ))` syntax to inject secrets, you need to create a `vars.yml` file locally. This file **should not** be committed to source control, as it contains sensitive information.

Below is an example of what `vars.yml` could look like:

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

1. **Create** a file named `vars.yml` in the same directory as your `manifest.yml`.
2. **Replace** the placeholder values (`YOUR_ACCESS_KEY_ID`, `YOUR_SECRET_ACCESS_KEY`, etc.) with your real credentials.
3. **Do not** commit `vars.yml` to your git repository. It’s recommended to add `vars.yml` to `.gitignore`.

## 3. Configure `manifest.yml`

Below is an example `manifest.yml` for deploying your Flask app with the Python buildpack, plus the Gremlin sidecar. Adjust memory, paths, and environment variables as needed:

```yaml
---
applications:
  - name: s3-failure-flags-app            # App name in CF
    memory: 512M                          # Memory for the container
    disk_quota: 1G                        # Disk space for the container
    instances: 1                          # Number of instances

    buildpacks:
      - python_buildpack                  # Use Python buildpack for Flask

    path: .                               # Push contents of current directory

    command: null                         # Let the buildpack detect and use your Procfile

    env:
      S3_BUCKET: "commoncrawl"            # Custom environment variables
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
        process_types: ["web"]            # Attach sidecar to 'web' process
        memory: 256M
        disk_quota: 256M
        command: "./failure-flags-sidecar"  # Run Gremlin sidecar
```

## 4. Push to Cloud Foundry

From your project directory:
```bash
cf push --vars-file vars.yml
```

- The Python buildpack installs dependencies (from `requirements.txt`: `flask`, `boto3`, `requests`, `failureflags`).
- Both the Flask app and the Gremlin sidecar run in the same container, allowing local communication (`localhost:5032`).

## Reference

- [Pushing apps with sidecar processes](https://docs.cloudfoundry.org/devguide/sidecars.html)
- [Attribute Reference - sidecars](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#sidecars)
