# Production

This page includes helpful tips for configuring things in production:

## Key management

The default production key management solution is [Google Cloud KMS][gcp-kms].
If you are using the Terraform configurations, the system will automatically
bootstrap and create the key rings and keys in Cloud KMS. If you are not using
the Terraform configurations, follow this guide to create the keys manually:

1.  Create a Google Cloud KMS key ring

    ```sh
    gcloud kms keyrings create "en-verification" \
      --location "us"
    ```

    Note that the "us" location is configurable. If you choose a different
    location, substitute it in all future commands.

1.  Create two signing keys - one for tokens and one for certificates:

    ```sh
    gcloud kms keys create "token-signing" \
      --location "us" \
      --keyring "en-verification" \
      --purpose "asymmetric-signing" \
      --default-algorithm "ec-sign-p256-sha256" \
      --protection-level "hsm"
    ```

    ```sh
    gcloud kms keys create "certificate-signing" \
      --location "us" \
      --keyring "en-verification" \
      --purpose "asymmetric-signing" \
      --default-algorithm "ec-sign-p256-sha256" \
      --protection-level "hsm"
    ```

    Note the "us" location is configurable, but the key purpose and algorithm
    must be the same as above.

1.  Create an encryption key for encrypting values in the database:

    ```sh
    gcloud kms keys create "database-encrypter" \
      --location "us" \
      --keyring "en-verification" \
      --purpose "encryption" \
      --rotation-period "30d" \
      --protection-level "hsm"
    ```

1.  Get the resource names to the keys:

    ```sh
    gcloud kms keys describe "token-signing" \
      --location "us" \
      --keyring "en-verification"
    ```

    ```sh
    gcloud kms keys describe "certificate-signing" \
      --location "us" \
      --keyring "en-verification"
    ```

    ```sh
    gcloud kms keys describe "database-encrypter" \
      --location "us" \
      --keyring "en-verification"
    ```

1.  Provide these values as the `TOKEN_SIGNING_KEY`, `CERTIFICATE_SIGNING_KEY`,
    and `DB_ENCRYPTION_KEY` respectively in the environment where the services
    will run. You also need to grant the service permission to use the keys.


## Observability (tracing and metrics)

The observability component is responsible for metrics. The following
configurations are available:

| Name                    | `OBSERVABILITY_EXPORTER` value  | Description
| ----------------------- | ------------------------------- | -----------
| OpenCensus Agent        | `OCAGENT`                       | Use OpenCensus.
| Stackdriver\*           | `STACKDRIVER`                   | Use Stackdriver.


## User administration

There are three types of "users" for the system:

-   **System administrator** - global system administrators are the IT
    administrators of the system. They can create new realms and edit global
    system configuration. System admins, however, do not have permissions to
    administer codes or perform realm-specific tasks beyond their creation.
    Typically a system administrator creates a realm, adds the initial realm
    admin, then removes themselves from the realm. To create a system
    administrator, use the `cmd/add-users` tool. There is presently no UI for
    adding a system administrator.

-   **Realm administrator** - realm administrators control the configuration of
    one or more realms. A user may be an administrator of 0 or more realms. If a
    user is an administrator of a realm, they have permissions to change
    realm-specific settings including the realm name, signing keys, SMS
    configuration, etc. To create a realm administrator, check the "Admin" box
    in the UI for a new or existing user to promote them to a realm
    administrator. Note that realm administration is per-realm; making a user an
    admin of "Realm 1" does not make them an admin of "Realm 2".

-   **User** - users have the ability to generate and lookup the status of
    codes. A user can be a member of multiple realms. To create a user, visit
    the realm and click "Add User". If a user is a member of multiple realms (by
    email address), they will be prompted to choose a realm after authenticating
    to the system.


## Rotating secrets

This section describes how to rotate secrets in the system.

### Cookie keys

**Recommended frequency:** 30 days, on breach

The cookie keys are an array. The items at odd indicies are HMAC keys and the
items at even indicies are encryption keys. The HMAC key should be 64 bytes and
the encryption key should be 32. Even though the array is flat, each even/odd
pairing is actually a tuple:

```text
[<hmac_key_1>, <encryption_key_1>, <hmac_key_2>, <encryption_key_2>]
```

Each key is supplied to this system as base64, for example:

```sh
# "<base64_hmac_key_1>,<base64_encryption_key_1>"
export COOKIE_KEYS="ARLaFwAqBGIkm5pLjAveJuahtCnX2NLoAUz2kCZKrScUaUkEaxHSvJLVYb5yAPCc441Cho5n5yp8jdEmy6hyig==,RLjcRZeqc07s6dh3OK4CM1POjHDZHC+usNU1w/XNTjM="
```

To rotate the cookie keys, generate two new keys of the correct lengths as
specificed above and insert them **into the front** of the array. **Do not
remove the existing values from the array**, as doing so will invalidate all
existing sessions.

```text
[<NEW_HMAC_KEY>, <NEW_ENCRYPTION_KEY>, <hmac_key_1>, <encryption_key_1>, <hmac_key_2>, <encryption_key_2>]
```

Just as before, the new values should be base64-encoded:

```sh
# "<base64_hmac_key_1>,<base64_encryption_key_1>,<base64_hmac_key_2>,<base64_encryption_key_2>"
export COOKIE_KEYS="c8+OD0vpvT/FrtGAtHc1nYhtkYMhjEEHCLgzuIiKJbskAbMI7bJxSnlBMKmc2AQmo8QVAViJuKoopuSuXE7tYw==,KRN9OK/lcs/uBWKQ2/1I0g9KR/iL3/MHuCn6esI02fs=,ARLaFwAqBGIkm5pLjAveJuahtCnX2NLoAUz2kCZKrScUaUkEaxHSvJLVYb5yAPCc441Cho5n5yp8jdEmy6hyig==,RLjcRZeqc07s6dh3OK4CM1POjHDZHC+usNU1w/XNTjM="
```

Upon deploying, all _new_ sessions will use these new keys. Old sessions will be
automatically upgraded on the next visit. After a period of time that you deem
acceptable (e.g. 30d), you can remove the older keys from the end of the array.

You can use `openssl` or similar tooling to generate the secret. Note that this
is **not** preferred since it requires a human to see the plaintext secret.

```sh
openssl rand -base64 64 | tr -d "\n" # or 32
```

If you are using a secret manager, put this value in the secret manager and
provide its _reference_ in the environment. If you are not using a secret
manager, provide this value directly in the environment.


### Cross-site request forgery (CSRF) keys

**Recommended frequency:** 90 days, on breach

To rotate the key, generate a new 32-byte key. You can use `openssl` or similar:

```sh
openssl rand -base64 32 | tr -d "\n"
```

Update the `CSRF_AUTH_KEY` environment variable and re-deploy. The system [only
supports a single key for CSRF](https://github.com/gorilla/csrf/issues/65). When
you deploy the new key, any existing open HTML forms will fail to submit as an
invalid request.


### Database encryption keys

**Recommend frequency:** 30 days, on breach

These keys control application-layer encryption of secrets before they are
stored in the database. For example, this key encrypts Twilio credentials so
they are not in plaintext in the database. Note that we only use the encryption
key where encryption is appropriate. For API keys and tokens, we HMAC the values
as their plaintext values are not required.

To rotate the encryption keys, rotate them in the underlying key manager. Note
that old entries will still be encrypted with the old key. You do not need to
upgrade them so long as the older key version is still available in your key
manager.

While unlikely, this may require you to update the `DB_ENCRYPTION_KEY`
environment variable.


### API Key signature HMAC keys

**Recommended frequency:** 90 days

This key is used as the HMAC secret when signing API keys. API keys are signed
and verified using this value. Like cookies, it accepts an array of values. The
first item in the array is used to sign all new API keys, but all remaining
values are still accepted as valid. These keys should be at least 64 bytes, but 128 is recommended.

To generate a new key:

```sh
openssl rand -base64 128 | tr -d "\n"
```

Insert this new value **into the front** of the `DB_APIKEY_SIGNATURE_KEY`
environment variable:

```sh
DB_APIKEY_SIGNATURE_KEY="gSEGlr482MSTm0eGRm2VvS86iQin3+/+80ALBkKKBYgu2EJyhGkvi8BULeFQDW/qZp2y3IbKY0MUTqioG7InBZdCkisYjr8UNuA+PONxMSaw/x8m+CXF28qb2WF0OGYQIPgbOdQ7cChG3Ox5AQgWFmNwyr486lTxSM8TE7dfCfU=,oXrnYzt6MXKBB/+zZWTvkUABW8SSCAFv5Mc475sSVPGBqCf1hWvv/VmByFj/5457Ho0AVbDUiCpKnjW2Q8ZlxqRo5dJyRifwvmW2JYcpxe+Ff/d+tb2x+TwlzqEzrKVatEWX4cmMG7ZP6B1oTCw/uZPTDhgB3lerXVIBTxdAaQc="
```

Note: Removing any of the keys from this list will invalidate API keys signed by
that version.


### API Key database HMAC keys

**Recommended frequency:** 90 days

This key is used as the HMAC secret when saving hashed API keys in the database.
Like cookies, it accepts an array of values. The first item in the array is used
to HMAC all new API keys, but all remaining values are still accepted as valid.
These keys should be at least 64 bytes, but 128 is recommended.

To generate a new key:

```sh
openssl rand -base64 128 | tr -d "\n"
```

Insert this new value **into the front** of the `DB_APIKEY_DATABASE_KEY`
environment variable:

```sh
DB_APIKEY_SIGNATURE_KEY="1do5HM96Bk9WD15BQC3qbW9e3T2V6T0DHn2i1xGJRKX8tZubxuaezivMhO3uJFEye8SITH3UFB+mo9oE0VGPiP/4TOXejfsd1g2L518itJbrK4/qNh6QMk0I04mqNtR55uvyt7G/ObADn2hQDYMVOGg/C6nLiqO+nqQ/UoUcGkA=,tJiUPEi0xS5QbykypVlquWsxQ0DAgxY41w+PkNqpoqzWQyDnEUAWFwIFUUFllqT+m0f2Kqh8EB1zjYgFcGP16O52rHer5sr4x6nsnQ/AiOHDrztJnEqGvutetHhZHLGKY0HxlxkZxcFLCmbgs6pa0vNUodrzOsCYysD7MLCSL5M="
```

Note: Removing any of the keys from this list will invalidate API keys HMACed by
that version.


### Verification Code database HMAC keys

**Recommended frequency:** 30 days

This key is used as the HMAC secret when saving verification codes in the
database. Like cookies, it accepts an array of values. The first item in the
array is used to HMAC all new verification codes, but all remaining values are
still accepted as valid. These keys should be at least 64 bytes, but 128 is
recommended.

To generate a new key:

```sh
openssl rand -base64 128 | tr -d "\n"
```

Insert this new value **into the front** of the `DB_VERIFICATION_CODE_DATABASE_KEY`
environment variable:

```sh
DB_APIKEY_SIGNATURE_KEY="g7GdsjuN+eydQIUCena2gleSHsmu46Gs+62ENViXsaV123AoVEwZ94ywpCQ2hxJ6CSBc4wXOwrxhg+psiwfp9eyBcpOFC7i98mOTLu1gxznZe943PVKl9vKJx9SgFrSnI1prWoQj85xGJKMBlM/pj608LWpZ3aIxyk0t7Sk/iWE=,G1VCqQVqe+4GD60YsqOHVgYEXN6WMh8tuF9bAfRJyt9sVk9kBWbPdhFQVUdCqoE3cckSsxz7LMApiN1/2jbwG3qkTicx4YuwQMUDVg2Stdv0L2kvek/+MYcA0lVYaNZWBJCSgmaMzjzOGW/BsbR/ssX1WhGI9aVoGpFQMiEJaVE="
```

Note: Removing any of the keys from this list will invalidate verification codes
HMACed by that version. However, given verification a verification code's
lifetime is short, it is probably safe to remove the key beyond 30 days.


### Certificate and token signing keys

**Recommended frequency:** on demand

If you are using system keys, the system administrator will handle rotation. If
you are using realm keys, you can generate new keys in the UI.

[gcp-kms]: https://cloud.google.com/kms
