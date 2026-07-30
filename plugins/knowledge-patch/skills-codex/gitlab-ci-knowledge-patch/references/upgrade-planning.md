# GitLab Upgrade Planning

Use this reference when preparing or recovering a GitLab upgrade. Separate
requirements by installation method before changing the database, registry,
operating system, or bundled services.

## Required GitLab 19 stops

The required upgrade stops are `19.2`, `19.5`, `19.8`, and `19.11`. Review the
notes for every version between the installed and target versions, including
installation-method-specific notes.

## Avoid the 19.2.0 Duo endpoint reset

A direct Linux package upgrade to `19.2.0` can clear the configured local AI
Gateway and Duo Agent Platform service URLs and reset related settings.

- Upgrade directly to `19.2.1` or later.
- If `19.2.0` has already cleared the settings, restore them under **Admin
  area** > **GitLab Duo** > **Configuration** > **Service endpoints**.

## Upgrade PostgreSQL first

GitLab 19.0 requires PostgreSQL 17 for every installation method. Upgrade a
packaged PostgreSQL 16 server or an external PostgreSQL deployment before
installing GitLab 19.

## Prepare the container registry

### Metadata database default and 19.0 route failures

For an existing Linux package or self-compiled installation with no explicit
`registry['database']['enabled']` value, GitLab 19.0 defaults the registry
metadata database to `prefer`. This mode falls back to legacy filesystem
metadata for data that has not been imported.

In `19.0.0` and `19.0.1`, this can make `/gitlab/v1/` routes return HTTP 500.
The `/v2/` image push and pull routes continue to work. Apply this temporary
override:

```ruby
registry['database'] = {
  'enabled' => false
}
```

Reconfigure and restart the registry. After upgrading to `19.0.2` or later,
remove the override.

### Move S3 storage to `s3_v2`

GitLab 19.0 removes the legacy AWS SDK v1 `s3` driver and aliases `s3` to
`s3_v2`. Update the configuration explicitly:

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

For non-AWS S3-compatible storage:

- `regionendpoint` must be a complete URI.
- Set `checksum_disabled` when the backend rejects enhanced upload checksums.
- Deletion still sends CRC32. The backend must support it; there is no GitLab
  configuration workaround for deletion.

## Repair Geo OCI image-index replication

Geo secondaries on `19.0.0` and `19.0.1` can silently omit OCI image-index
tags, including multi-architecture images and BuildKit cache tags.

1. Upgrade both the primary and secondary sites to `19.0.2` or later.
2. Let existing repositories recover during verification, which can take up to
   the default 90-day interval, or manually resync their container repositories
   for immediate repair.

## Migrate unsupported package platforms

### Ubuntu 20.04

GitLab 18.11 is the last release with Linux packages for Ubuntu 20.04. Move to
Ubuntu 22.04 or another supported operating system before GitLab 19.

### SUSE

GitLab 18.11 is the last release providing Linux packages for openSUSE Leap
15.6, SLES 12.5, and SLES 15.6. Installations that must remain on SUSE for
GitLab 19 need to migrate to Docker.

## Replace removed data and application services

### Redis 6

GitLab 19.0 removes support for Redis 6. Migrate external deployments to Redis
7.0 or later or Valkey 7.2 before upgrading. The Redis bundled with the Linux
package is already version 7 and is unaffected.

### Bundled Mattermost

GitLab 19.0 removes Mattermost from the Linux package.

1. Migrate users of the bundled service to standalone Mattermost.
2. Remove or comment out every `mattermost[...]` key in
   `/etc/gitlab/gitlab.rb`.
3. Perform the upgrade and reconfigure.

If the keys remain, `gitlab-ctl reconfigure` aborts.
`gitlab-ctl check-config --version 19.0.x` does not detect this problem.

### Bundled Spamcheck

GitLab 19.0 removes bundled Spamcheck from both the Linux package and Helm
chart. Existing users must deploy it separately, such as with Docker. No data
migration is required.

## Prepare Helm and Operator installations

### Gateway API and Envoy Gateway

The GitLab 19.0 Helm chart defaults to Gateway API with Envoy Gateway instead
of NGINX Ingress.

- The bundled NGINX Ingress can be explicitly re-enabled until its proposed
  removal in GitLab 20.0.
- Externally managed Ingress controllers, externally managed Gateway API
  controllers, and Linux package NGINX are unaffected.

### External data services

GitLab 19.0 removes the bundled Bitnami PostgreSQL, Bitnami Redis, and MinIO
charts from the GitLab Helm chart and Operator without replacements. Configure
external services before upgrading an installation that used these
proof-of-concept components.

## Clean orphaned RPM directories

RPM installations of `19.0.0` through `19.0.2` and `19.1.0` can leave
`.agents` and `.claude` under
`/opt/gitlab/embedded/service/gitlab-rails/`. RPM does not remove these
nonempty directories after it stops owning them.

After reaching `19.0.3`, `19.1.1`, or `19.2` and later, check for and manually
remove those exact directories. DEB installations are unaffected.
