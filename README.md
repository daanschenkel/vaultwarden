# Fork from [dani-garcia/vaultwarden](https://github.com/dani-garcia/vaultwarden)

Goal is to help testing code for the SSO [PR](https://github.com/dani-garcia/vaultwarden/pull/3899).
Based on [Timshel/sso-support](https://github.com/Timshel/vaultwarden/tree/sso-support)

:warning: Branch will be rebased and forced-pushed from time to time. :warning:

## Docker

Change the docker files to package both front-end from [Timshel/oidc_web_builds](https://github.com/Timshel/oidc_web_builds/releases).
\
By default it will use the release which only make the `sso` button visible.

If you want to use the version which additionally change the default redirection to `/sso` and fix organization invitation to persist.
You need to pass an env variable: `-e SSO_FRONTEND='override'` (cf [start.sh](docker/start.sh)).

## To test VaultWarden with Keycloak

[Readme](test/oidc/README.md)

## DB Migration

ATM The migrations add an independant table `sso_nonce` and a column `invited_by_email` to `users_organizations`.

### Revert to default VW

Reverting to the default VW DB state can easily be done manually (Make a backup :) :

```psql
>BEGIN;
BEGIN
>DELETE FROM __diesel_schema_migrations WHERE version in ('20230910133000', '20230914133000');
DELETE 2
>DROP TABLE sso_nonce;
DROP TABLE
>ALTER TABLE users_organizations DROP COLUMN invited_by_email;
ALTER TABLE
> COMMIT / ROLLBACK;
```

### FROM old PR Version

:warning: Changed the past migration creating the `sso_nonce` table in a recent [commit](https://github.com/Timshel/vaultwarden/commit/afa26f3cf5a39ff0bc4c3cbe563cfcfaf91b40a0).:warning: <br>
If you already deployed the previous version you'll need to do some manual cleanup :

```psql
>BEGIN;
BEGIN
>DELETE FROM __diesel_schema_migrations WHERE version = '20230201133000';
DELETE 1
>DROP TABLE sso_nonce;
DROP TABLE
> COMMIT / ROLLBACK;
```

Then the new migration will play without issue.
.
