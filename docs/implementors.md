# Implementors

This page contains a list of projects leveraging ORAS, as well as
registries that are known to support [OCI Artifacts][artifacts].

*Would like your registry and/or project listed here?
Please [submit a PR](https://github.com/oras-project/oras-www/pulls).
We're happy to promote all usage, as well as provide feedback.*

## Projects using ORAS

- [Helm](https://v3.helm.sh/docs/topics/registries/)
- [Singularity](https://sylabs.io/guides/3.1/user-guide/cli/singularity_push.html)

## Registries supporting OCI Artifacts

- [CNCF Distribution](#cncf-distribution) - local/offline verification
- [Azure Container Registry](#azure-container-registry-acr)
- [Amazon Elastic Container Registry](#amazon-elastic-container-registry-ecr)
- [Google Artifact Registry](#google-artifact-registry-gar)
- [GitHub Packages container registry](#github-container-registry)
- [Bundle Bar](#bundle-bar)

### CNCF Distribution

[https://github.com/distribution/distribution](https://github.com/distribution/distribution) version 2.7+

[CNCF Distribution](https://github.com/distribution/distribution) is a reference implementation of the [OCI distribution-spec][distribution-spec]. Running distribution locally, as a container, provides local/offline verification of ORAS and [OCI Artifacts][artifacts].

#### Using a local, unauthenticated container registry

Run the [docker registry image](https://hub.docker.com/_/registry) locally:

```
docker run -it --rm -p 5000:5000 registry
```

This will start a distribution server at `localhost:5000`
*(with wide-open access and no persistence outside of the container)*.

#### Using Docker Registry with authentication

- Create a valid htpasswd file (must use `-B` for bcrypt):

  ```
  htpasswd -cB -b auth.htpasswd myuser mypass
  ```

- Start a registry using the password file for authentication:

  ```
  docker run -it --rm -p 5000:5000 \
      -v $(pwd)/auth.htpasswd:/etc/docker/registry/auth.htpasswd \
      -e REGISTRY_AUTH="{htpasswd: {realm: localhost, path: /etc/docker/registry/auth.htpasswd}}" \
      registry
  ```

- In a new window, login with `oras`:

  ```
  oras login -u myuser -p mypass localhost:5000
  ```

You will notice a new entry for `localhost:5000` appear in `~/.docker/config.json`.

To remove the entry from the credentials file, use `oras logout`:

```
oras logout localhost:5000
```

#### Using an insecure Docker registry

To login to the registry without a certificate, a self-signed certificate, or an unencrypted HTTP connection Docker registry, `oras` supports the `--insecure` flag.

- Create a valid htpasswd file (must use `-B` for bcrypt):

  ```
  htpasswd -cB -b auth.htpasswd myuser mypass
  ```

- Generate your self-signed certificates:

  ```
  $ mkdir -p certs
  $ openssl req \
    -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
    -x509 -days 365 -out certs/domain.crt
  ```

- Start a registry using that file for auth and listen the `0.0.0.0` address:

  ```
  docker run -it --rm -p 5000:5000 \
      -v `pwd`/certs:/certs \
      -v $(pwd)/auth.htpasswd:/etc/docker/registry/auth.htpasswd \
      -e REGISTRY_AUTH="{htpasswd: {realm: localhost, path: /etc/docker/registry/auth.htpasswd}}" \
      -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
      -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
      registry
  ```

- In a new window, login with `oras` using the ip address not localhost:

  ```
  oras login -u myuser -p mypass --insecure <registry-ip>:5000
  ```

You will notice a new entry for `<registry-ip>:5000` appear in `~/.docker/config.json`.

Then you can pull files from the registry or push files to the registry.

- To push single file to this registry:

  ```
  oras push <registry-ip>:5000/library/hello:latest hi.txt --insecure
  ```

- To pull files from this registry:

  ```
  oras pull <registry-ip>:5000/library/hello:latest --insecure
  ```

- To remove the entry from the credentials file, use `oras logout`:

  ```
  oras logout <registry-ip>:5000
  ```

##### Using a plain HTTP Docker registry

To pull or push the HTTP Docker registry. `oras` support `--plain-http` flag to pull or push.

The `--plain-http` flag mean that you want to use http instead of https to connect the Docker registry.

- Create a valid htpasswd file (must use `-B` for bcrypt):

  ```
  htpasswd -cB -b auth.htpasswd myuser mypass
  ```

- Start a registry using that file for auth and listen the `0.0.0.0` address:

  ```
  docker run -it --rm -p 5000:5000 \
    -v $(pwd)/auth.htpasswd:/etc/docker/registry/auth.htpasswd \
    -e REGISTRY_AUTH="{htpasswd: {realm: localhost, path: /etc/docker/registry/auth.htpasswd}}" \
    -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
    registry
  ```

- In a new window, login with `oras` using the ip address not localhost:

  ```
  oras login -u myuser -p mypass --insecure <registry-ip>:5000
  ```

You will notice a new entry for `<registry-ip>:5000` appear in `~/.docker/config.json`.

Then you can pull files from the registry or push files to the registry.

- To push single file to this registry:

  ```
  oras push <registry-ip>:5000/library/hello:latest hi.txt --plain-http
  ```

- To pull files from this registry:

  ```
  oras pull <registry-ip>:5000/library/hello:latest --plain-http
  ```

- To remove the entry from the credentials file, use `oras logout`:

  ```
  oras logout <registry-ip>:5000
  ```

#### [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)

ECR Artifact Blog Post: <https://aws.amazon.com/blogs/containers/oci-artifact-support-in-amazon-ecr/>

- Authenticating with ECR using the AWS CLI
  ```
  aws ecr get-login-password --region $AWS_REGION --profile $PROFILE | oras login \
      --password-stdin \
      --username AWS \
      "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
  ```

- Pushing Artifacts to ECR

  ```
  oras push $REPO_URI:1.0 \
      --manifest-config /dev/null:application/vnd.unknown.config.v1+json \
      ./artifact.txt:application/vnd.unknown.layer.v1+txt
  ```

- Pulling Artifacts from ECR

  ```
  oras pull $REPO_URI:1.0 \
    --media-type application/vnd.unknown.layer.v1+txt
  ```

#### [Azure Container Registry (ACR)](https://aka.ms/acr)

ACR Artifact Documentation: [aka.ms/acr/artifacts](https://aka.ms/acr/artifacts)

- Authenticating with ACR using [Service Principals](https://docs.microsoft.com/azure/container-registry/container-registry-auth-service-principal)

  ```
  oras login myregistry.azurecr.io --username $SP_APP_ID --password $SP_PASSWD
  ```

- Authenticating with ACR [using AAD credentials](https://docs.microsoft.com/azure/container-registry/container-registry-authentication) and the [`az cli`](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest)

  ```
  az login
  az acr login --name myregistry
  ```

- Pushing Artifacts to ACR

  ```
  oras push myregistry.azurecr.io/samples/artifact:1.0 \
      --manifest-config /dev/null:application/vnd.unknown.config.v1+json \
      ./artifact.txt:application/vnd.unknown.layer.v1+txt
  ```

- Pulling Artifacts from ACR

  ```
  oras pull myregistry.azurecr.io/samples/artifact:1.0 \
    --media-type application/vnd.unknown.layer.v1+txt
  ```

#### [Google Artifact Registry (GAR)](https://cloud.google.com/artifact-registry)

- Authenticating with GAR using the gcloud command-line tool

  ```
  gcloud auth configure-docker ${REGION}-docker.pkg.dev
  ```

- Pushing Artifacts to GAR

  ```
  oras push ${REGION}-docker.pkg.dev/${GCP_PROJECT}/samples/artifact:1.0 \
    ./artifact.txt:application/vnd.unknown.layer.v1+txt
  ```

- Pulling Artifacts from GAR

  ```
  oras pull ${REGION}-docker.pkg.dev/${GCP_PROJECT}/samples/artifact:1.0 \
    --media-type application/vnd.unknown.layer.v1+txt
  ```

#### [GitHub Packages container registry (GHCR)](https://docs.github.com/en/packages/guides/about-github-container-registry)

- [Authenticating with GHCR](https://docs.github.com/en/packages/guides/pushing-and-pulling-docker-images#authenticating-to-github-container-registry)

  ```
  echo $GITHUB_PAT | oras login https://ghcr.io -u GITHUB_USERNAME --password-stdin
  ```

- Pushing Artifacts to GHCR

  ```
  oras push ghcr.io/${GITHUB_OWNER}/samples/artifact:1.0 \
    ./artifact.txt:application/vnd.unknown.layer.v1+txt
  ```

- Pulling Artifacts from GHCR

  ```
  oras pull ghcr.io/${GITHUB_OWNER}/samples/artifact:1.0 \
    --media-type application/vnd.unknown.layer.v1+txt
  ```

#### [Bundle Bar](https://bundle.bar/docs/supported-clients/oras/)

- Authenticating with Bundle Bar

  ```
  echo $BB_TOKEN | oras login bundle.bar -u $BB_USER --password-stdin
  ```

- Pushing Artifacts to Bundle Bar

  ```
  oras push bundle.bar/u/${BB_USER}/samples/artifact:1.0 \
    ./artifact.txt:application/vnd.unknown.layer.v1+txt
  ```

- Pulling Artifacts from Bundle Bar

  ```
  oras pull bundle.bar/u/${BB_USER}/samples/artifact:1.0 \
    --media-type application/vnd.unknown.layer.v1+txt
  ```

[artifacts]:            https://github.com/opencontainers/artifacts
[distribution-spec]:    https://github.com/opencontainers/distribution-spec/