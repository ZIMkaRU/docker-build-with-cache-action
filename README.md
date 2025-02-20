# Docker build-with-cache action

This action builds your docker image and caches the stages (supports multi-stage builds) to improve building times in subsequent builds.

By default, it pushes the image with all the stages to a registry (needs username and password), but you can disable this feature by setting `push_image_and_stages` to `false`.

Built-in support for the most known registries:

- Docker Hub
- AWS ECR (private and public)
- GitHub's (old and new registry)
- Google Cloud's

## Inputs

### Required

- **image_name**: Image name (e.g. *node*).

or

- **compose_file**: path to Docker Compose file. You will need to configure this action multiple times if you have a compose file which uses more than one registry.

> :star2: New in v5.10.0: Now you can use [overrides](https://docs.docker.com/compose/extends/#multiple-compose-files) for your compose file(s) like this:  
  `docker-compose.yml > docker-compose.override.yml > docker-compose.override2.yml`

### Optional

- **image_tag**: Tag(s) of the image. Allows multiple comma-separated tags (e.g. `one,another`) (default: `latest`).  
  If you set **compose_file** and the image(s) already has/have a tag, this is ignored.

- **registry**: Docker registry (default: *Docker Hub's registry*). You need a registry to use the cache functionality.

- **username**: Docker registry's user (needed to push images, or to pull from a private repository).

- **password**: Docker registry's password (needed to push images, or to pull from a private repository).

- **session**: Extra auth parameters. For AWS ECR, means setting AWS_SESSION_TOKEN environment variable.

- **push_git_tag**: In addition to `image_tag`, you can also push the git tag in your [branch tip][branch tip] (default: `false`).

- **pull_image_and_stages**: Set to `false` to avoid pulling from the registry or to build from scratch (default: `true`).

- **stages_image_name**: Set custom name for stages. Useful if using a job matrix (default: `$image_name-stages)`.

- **build_extra_args**: Extra params for `docker build` (e.g. `"--build-arg=hello=world"`).  
  > :star2: New in v5.11.0: If you need extra args with newlines or spaces, use json format like this:  
    `build_extra_args: '{"--build-arg": "myarg=Hello\nWorld"}'`

- **push_image_and_stages**: Test a command before pushing. Use `false` to not push at all (default: `true`).

    This input also supports 2 special values, which are useful if your workflow can be triggered by different events:

    - `on:push`: Push only if the workflow was triggered by a push.
    - `on:pull_request`: Push only if the workflow was triggered by a pull_request.

[branch tip]: https://stackoverflow.com/questions/16080342/what-is-a-branch-tip-in-git

#### Ignored if `compose_file` is set

- **image_name**

- **context**: Docker context (default: `./`).

- **dockerfile**: Dockerfile filename path (default: `"$context"/Dockerfile`).

## Outputs

- **FULL_IMAGE_NAME**: Full name of the Docker Image with the Registry (if provided) and Namespace included.  
e.g.: `docker.pkg.github.com/whoan/hello-world/hello-world`

## How it works

The action does the following every time it is triggered:

- (Optional) Pull previously pushed [stages](https://docs.docker.com/develop/develop-images/multistage-build/) (if any) from the specified `registry` (default: https://hub.docker.com)
- Build the image using cache (i.e. the pulled stages)
- Tag the image
- (Optional) Push the image with the tag(s) specified in `image_tag`
- (Optional) Push each stage to the registry with names like `<image_name>-stages:<1,2,3,...>`
- (Optional) Push the git tag as `<image_name>:<git_tag>` if you set `push_git_tag: true`

## Examples

Find working minimal examples for the most known registries in [this repo](https://github.com/whoan/hello-world/tree/master/.github/workflows).

### Docker Hub

> If you don't specify a registry, Docker Hub is the default one

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: whoan
    password: "${{ secrets.DOCKER_HUB_PASSWORD }}"
    image_name: hello-world
```

### GitHub Registry

> [GitHub automatically creates a GITHUB_TOKEN secret to use in your workflow](https://help.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token#about-the-github_token-secret). If you are going to use the new GitHub Registry (ghcr.io), be sure to use a Personal Access Token (as the password) with "write:packages" and "read:packages" scopes. More info [here](https://docs.github.com/en/packages/getting-started-with-github-container-registry/migrating-to-github-container-registry-for-docker-images#migrating-a-docker-image-using-the-docker-cli).

> If you push the image to a **public** repository's GitHub Registry, please be aware that it will be impossible to delete it because of GitHub's policy (see [Deleting a package](https://help.github.com/en/packages/publishing-and-managing-packages/deleting-a-package)).

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: whoan
    password: "${{ secrets.GITHUB_TOKEN }}"
    registry: docker.pkg.github.com
    #or
    #registry: ghcr.io
    image_name: hello-world
```

### Google Cloud Registry

> More info [here](https://cloud.google.com/container-registry/docs/advanced-authentication#json-key) on how to get GCloud JSON key.

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: _json_key
    password: "${{ secrets.GCLOUD_JSON_KEY }}"
    registry: gcr.io
    image_name: hello-world
```

### AWS ECR

> You don't even need to create the repositories in advance, as this action takes care of that for you!

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: "${{ secrets.AWS_ACCESS_KEY_ID }}"  # no need to provide it if you already logged in with aws-actions/configure-aws-credentials
    password: "${{ secrets.AWS_SECRET_ACCESS_KEY }}"  # no need to provide it if you already logged in with aws-actions/configure-aws-credentials
    session:  "${{ secrets.AWS_SESSION_TOKEN }}"  # if you need role assumption
    # private registry
    registry: 861729690598.dkr.ecr.us-west-1.amazonaws.com
    # or public registry
    #registry: public.ecr.aws
    image_name: hello-world
```

### From a compose file

> Images to build are detected `grep`ping the registry (if provided) in the compose file. If no registry is provided, DockerHub is assumed and images with format `<username>/<image>[:<tag>]` are `grep`ped.

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: whoan
    password: "${{ secrets.DOCKER_HUB_PASSWORD }}"
    compose_file: docker-compose.yml
```

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: whoan
    password: "${{ secrets.GITHUB_TOKEN }}"
    registry: docker.pkg.github.com
    compose_file: docker-compose.yml
```

With a compose file override:

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: whoan
    password: "${{ secrets.DOCKER_HUB_PASSWORD }}"
    compose_file: docker-compose.yml > docker-compose.override.yml
```

### Example with more options

```yml
- uses: whoan/docker-build-with-cache-action@v5
  with:
    username: whoan
    password: "${{ secrets.GITHUB_TOKEN }}"
    image_name: whoan/docker-images/node
    image_tag: alpine-slim,another-tag,latest
    push_git_tag: true
    registry: docker.pkg.github.com
    context: node-alpine-slim
    dockerfile: custom.Dockerfile
    build_extra_args: "--compress=true --build-arg=hello=world"
    push_image_and_stages: docker run my_awesome_image:latest  # eg: push only if docker run succeed
```

## Cache is not working?

- Be specific with the base images. e.g.: if you start from an image with the `latest` tag, it may download different versions when the action is triggered, and it will invalidate the cache.
- If you are using Buildkit, the stages won't be pushed to the registry. This might be supported in a future version.
- Some docker limitations might cause the cache not to be used correctly. More information [in this SO answer](https://stackoverflow.com/questions/54574821/docker-build-not-using-cache-when-copying-gemfile-while-using-cache-from/56024061#56024061).

## Tests

The tests for this action are run in a [separate repo](https://github.com/whoan/hello-world) as I need to set credentials for each registry with GitHub secrets and doing so in this repo is not practical.

## License

[MIT](https://github.com/whoan/docker-build-with-cache-action/blob/master/LICENSE)
