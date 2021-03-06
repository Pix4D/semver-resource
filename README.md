# Build Number Resource

A resource for incrementing a counter that would represent
an internal build number.

## Source Configuration

- `initial_build_number`: _Optional._ The version number to use when
  bootstrapping, i.e. when there is not a version number present in the source.

- `driver`: _Optional. Default `s3`._ The driver to use for tracking the
  version. Determines where the version is stored.

There are three supported drivers, with their own sets of properties for
configuring them.

### `git` Driver

The `git` driver works by modifying a file in a repository with every bump. The
`git` driver has the advantage of being able to do atomic updates.

- `uri`: _Required._ The repository URL.

- `branch`: _Required._ The branch the file lives on.

- `file`: _Required._ The name of the file in the repository.

- `private_key`: _Optional._ The SSH private key to use when pulling from/pushing to to the repository.

- `username`: _Optional._ Username for HTTP(S) auth when pulling/pushing.
  This is needed when only HTTP/HTTPS protocol for git is available (which does not support private key auth)
  and auth is required.

- `password`: _Optional._ Password for HTTP(S) auth when pulling/pushing.

- `git_user`: _Optional._ The git identity to use when pushing to the
  repository support RFC 5322 address of the form "Gogh Fir \<gf@example.com\>" or "foo@example.com".

- `depth`: _Optional._ If a positive integer is given, shallow clone the repository using the --depth option.

- `commit_message`: _Optional._ If specified overides the default commit message with the one provided. The user can use %version% and %file% to get them replaced automatically with the correct values.

### `s3` Driver

The `s3` driver works by modifying a file in an S3 compatible bucket.

- `bucket`: _Required._ The name of the bucket.

- `key`: _Required._ The key to use for the object in the bucket tracking
  the version.

- `access_key_id`: _Required._ The AWS access key to use when accessing the
  bucket.

- `secret_access_key`: _Required._ The AWS secret key to use when accessing
  the bucket.

- `region_name`: _Optional. Default `us-east-1`._ The region the bucket is in.

- `endpoint`: _Optional._ Custom endpoint for using S3 compatible provider.

- `disable_ssl`: _Optional._ Disable SSL for the endpoint, useful for S3 compatible providers without SSL.

- `skip_ssl_verification`: _Optional._ Skip SSL verification for S3 endpoint. Useful for S3 compatible providers using self-signed SSL certificates.

- `server_side_encryption`: _Optional._ The server-side encryption algorithm
  used when storing the version object (e.g. `AES256`, `aws:kms`).

- `use_v2_signing`: _Optional._ Use v2 signing, default false.

### `swift` Driver

The `swift` driver works by modifying a file in a container.

- `openstack` _Required._ All openstack configuration must go under this key.

  - `container`: _Required._ The name of the container.

  - `item_name`: _Required._ The item name to use for the object in the container tracking
    the version.

  - `region`: _Required._ The region the container is in.

  - `identity_endpoint`, `username`, `user_id`, `password`, `api_key`, `domain_id`, `domain_name`, `tenant_id`, `tenant_name`, `allow_reauth`, `token_id`: See below
    The swift driver uses [gophercloud](http://gophercloud.io/docs/) to handle interacting
    with OpenStack. All OpenStack Identity versions are supported through this library. The
    Authentication properties will pass through to it. For detailed information about the
    individual parameters, see https://github.com/rackspace/gophercloud/blob/master/auth_options.go

### `gcs` Driver

The `gcs` driver works by modifying a file in a Google Cloud Storage bucket.

- `bucket`: _Required._ The name of the bucket.

- `key`: _Required._ The key to use for the object in the bucket tracking the version.

- `json_key`: _Required._ The contents of your GCP Account JSON Key. Example:

  ```yaml
  json_key: |
    {
      "private_key_id": "...",
      "private_key": "...",
      "client_email": "...",
      "client_id": "...",
      "type": "service_account"
    }
  ```

### Example

With the following resource configuration:

```yaml
resources:
- name: version
  type: semver
  source:
    driver: git
    uri: git@github.com:concourse/concourse.git
    branch: version
    file: version
    private_key: {{concourse-repo-private-key}}
```

Bumping with a `get` and then a `put`:

```yaml
plan:
- get: version
  params: {bump: minor}
- task: a-thing-that-needs-a-version
- put: version
  params: {file: version/version}
```

Or, bumping with an atomic `put`:

```yaml
plan:
- put: version
  params: {bump: minor}
- task: a-thing-that-needs-a-version
```

## Behavior

### `check`: Report the current version number.

Detects new versions by reading the file from the specified source. If the file is empty, it returns the `initial_build_number`. If the file is not empty, it returns the version specified in the file if it is equal to or greater than current version, otherwise it returns no versions.

### `in`: Provide the version as a file, optionally bumping it.

Provides the version number to the build as a `version` file in the destination.

Can be configured to bump the version locally, which can be useful for getting
the `final` version ahead of time when building artifacts.

#### Parameters

- `bump` and `pre`: _Optional._ See [Version Bumping
  Semantics](#version-bumping-semantics).

Note that `bump` and `pre` don't update the version resource - they just
modify the version that gets provided to the build. An output must be
explicitly specified to actually update the version.

### `out`: Set the version or bump the current one.

Given a file, use its contents to update the version. Or, given a bump
strategy, bump whatever the current version is. If there is no current version,
the bump will be based on `initial_build_number`.

The `file` parameter should be used if you have a particular version that you
want to force the current version to be. This can be used in combination with
`in`, but it's probably better to use the `bump` and `pre` params as they'll
perform an atomic in-place bump if possible with the driver.

#### Parameters

One of the following must be specified:

- `file`: _Optional._ Path to a file containing the version number to set.

- `bump` and `pre`: _Optional._ See [Version Bumping
  Semantics](#version-bumping-semantics).

When `bump` and/or `pre` are used, the version bump will be applied atomically,
if the driver supports it. That is, if we pull down version `N`, and bump to
`N+1`, the driver can then compare-and-swap. If the compare-and-swap fails
because there's some new version `M`, the driver will re-apply the bump to get
`M+1`, and try again (in a loop).

## Version Bumping Semantics

Both `in` and `out` support bumping the version semantically via two params:
`bump` and `pre`:

- `bump`: _Optional._ Bump the version number semantically. The value must
  be one of:

  - `major`: Bump the major version number, e.g. `1.0.0` -> `2.0.0`.
  - `minor`: Bump the minor version number, e.g. `0.1.0` -> `0.2.0`.
  - `patch`: Bump the patch version number, e.g. `0.0.1` -> `0.0.2`.
  - `final`: Promote the version to a final version, e.g. `1.0.0-rc.1` -> `1.0.0`.

* `pre`: _Optional._ When bumping, bump to a prerelease (e.g. `rc` or
  `alpha`), or bump an existing prerelease.

  If present, and the version is already a prerelease matching this value,
  its number is bumped. If the version is already a prerelease of another
  type, (e.g. `alpha` vs. `beta`), the type is switched and the prerelease
  version is reset to `1`. If the version is _not_ already a pre-release, then
  `pre` is added, starting at `1`.

### Running the tests

The tests have been embedded with the `Dockerfile`; ensuring that the testing
environment is consistent across any `docker` enabled platform. When the docker
image builds, the test are run inside the docker container, on failure they
will stop the build.

Run the tests with the following command:

```sh
docker build -t semver-resource .
```

#### Integration tests

The integration requires two AWS S3 buckets, one without versioning and another
with. The `docker build` step requires setting `--build-args` so the
integration will run.

You will need:

- AWS key and secret
- An S3 bucket
- The region you are in (i.e. `us-east-1`, `us-west-2`)

Run the tests with the following command, replacing each `build-arg` value with your own values:

```sh
docker build . -t semver-resource --build-arg SEMVER_TESTING_ACCESS_KEY_ID="some-key" --build-arg SEMVER_TESTING_SECRET_ACCESS_KEY="some-secret" --build-arg SEMVER_TESTING_BUCKET="some-bucket" --build-arg SEMVER_TESTING_REGION="some-region"
```

### Contributing

Please make all pull requests to the `master` branch and ensure tests pass
locally.
