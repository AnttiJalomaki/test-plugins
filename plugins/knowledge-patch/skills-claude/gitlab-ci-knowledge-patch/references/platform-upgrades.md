# GitLab Platform Upgrade Guidance

## Plan the upgrade path

GitLab 19 required upgrade stops are `19.2`, `19.5`, `19.8`, and `19.11`.
Visit every stop that lies between the installed and target versions. Also
review every intervening release note and the notes specific to the
installation method.

## Protect self-hosted service endpoints

A direct Linux package upgrade to 19.2.0 can clear local AI Gateway and Duo
Agent Platform service URLs and reset related settings. Upgrade directly to
19.2.1 or later.

If 19.2.0 already caused the reset, restore the endpoints under **Admin area**
> **GitLab Duo** > **Configuration** > **Service endpoints**.

## Prepare PostgreSQL and external key-value storage

GitLab 19.0 requires PostgreSQL 17 for every installation method. Upgrade a
packaged PostgreSQL 16 server or external PostgreSQL deployment before
installing GitLab 19.

Redis 6 is no longer supported. Move external deployments to Redis 7.0 or
later, or Valkey 7.2, before the GitLab upgrade. The Redis bundled in the Linux
package is already version 7 and does not need this migration.

## Handle the registry metadata database default

For an existing Linux package or self-compiled installation without an
explicit `registry['database']['enabled']` value, GitLab 19.0 defaults the
registry metadata database to `prefer` mode. This mode falls back to legacy
filesystem metadata when the database has not imported the data.

On 19.0.0 and 19.0.1, the fallback can make `/gitlab/v1/` routes return HTTP
500. `/v2/` image pushes and pulls continue to work. Recover by temporarily
disabling the database, reconfiguring, and restarting the registry:

```ruby
registry['database'] = {
  'enabled' => false
}
```

Upgrade to 19.0.2 or later, then remove the temporary override.

## Migrate container registry storage

GitLab 19.0 removes the AWS SDK v1 `s3` registry driver and aliases `s3` to
`s3_v2`. For a non-AWS S3-compatible backend:

- make `regionendpoint` a complete URI;
- enable path-style access if the backend requires it; and
- set `checksum_disabled` when enhanced upload checksums are rejected.

For example:

```ruby
registry['storage'] = {
  's3_v2' => {
    'accesskey' => '<your-access-key>',
    'secretkey' => '<your-secret-key>',
    'bucket' => '<your-bucket>',
    'region' => '<your-region>',
    'regionendpoint' => 'https://storage.example.com',
    'pathstyle' => true,
    'checksum_disabled' => true
  }
}
```

Deletion still sends CRC32 even with `checksum_disabled`. A backend that
rejects that checksum must add support; there is no GitLab configuration
workaround.

## Repair Geo OCI image-index replication

Geo secondaries on 19.0.0 and 19.0.1 can silently omit OCI image-index tags.
This includes multi-architecture image tags and BuildKit cache tags.

Upgrade both sites to 19.0.2 or later. Existing repositories recover during
verification, but that can take up to the default 90-day interval. Manually
resync affected container repositories when recovery cannot wait.

## Move off unsupported operating-system packages

GitLab 18.11 is the final Linux package release for Ubuntu 20.04. Move to
Ubuntu 22.04 or another supported operating system before upgrading to GitLab
19.

It is also the final package release for openSUSE Leap 15.6, SLES 12.5, and
SLES 15.6. Installations that must remain on those SUSE systems need to
migrate to a Docker deployment for GitLab 19.

## Remove bundled Mattermost configuration

GitLab 19.0 removes Mattermost from the Linux package. Migrate bundled-service
users to a standalone Mattermost installation, then remove or comment out
every `mattermost[...]` key in `/etc/gitlab/gitlab.rb`.

Do this before upgrading. Otherwise `gitlab-ctl reconfigure` aborts.
`gitlab-ctl check-config --version 19.0.x` does not detect the obsolete keys.

## Externalize Spamcheck

GitLab 19.0 removes bundled Spamcheck from both the Linux package and the Helm
chart. Existing users must deploy it separately, for example with Docker. No
Spamcheck data migration is required.

## Prepare the Helm networking transition

The GitLab 19.0 Helm chart defaults to Gateway API with Envoy Gateway instead
of NGINX Ingress. The bundled NGINX Ingress can be explicitly re-enabled until
its proposed removal in GitLab 20.0.

The change does not affect:

- an externally managed Ingress controller;
- an externally managed Gateway API controller; or
- the NGINX bundled in the Linux package.

## Externalize Helm chart data services

GitLab 19.0 removes the bundled Bitnami PostgreSQL, Bitnami Redis, and MinIO
charts from both the GitLab Helm chart and the Operator. There are no bundled
replacements. Configure external services before upgrading an installation
that used these proof-of-concept components.

## Clean orphaned RPM directories

RPM installs of 19.0.0 through 19.0.2 and 19.1.0 can leave nonempty `.agents`
and `.claude` directories under:

```text
/opt/gitlab/embedded/service/gitlab-rails/
```

RPM no longer owns those directories and therefore does not remove them.
After reaching 19.0.3, 19.1.1, or 19.2 and later, inspect and manually remove
these exact orphaned directories. DEB installations are unaffected.
