# README.fork.md — what this fork adds

This repository is a fork of
[`ministackorg/ministack`](https://github.com/ministackorg/ministack) kept at
`v1.5.40` plus one change that is not upstream yet: **S3 bucket state is
isolated per account and region**. [`README.md`](README.md) stays the upstream
document, so a rebase never has to resolve documentation conflicts and the
upstream pull request carries only the S3 work. This file holds everything that
is true of the fork and only the fork.

| | |
| --- | --- |
| Fork | [`mathewmusango/ministack-upstream`](https://github.com/mathewmusango/ministack-upstream) |
| Upstream | [`ministackorg/ministack`](https://github.com/ministackorg/ministack) |
| Branch | `feat/s3-region-scoped-state` — `main` is upstream + the CI workflow + this patch |
| Tag | `v1.5.40-fork.1`, image `ghcr.io/mathewmusango/ministack-upstream:1.5.40-fork.1-full` |
| Rebase | `git fetch upstream && git rebase upstream/main` |

## The difference: S3 is region-isolated

Upstream keeps S3 buckets in one account-scoped map, so every region of an
account shares one bucket namespace: `ListBuckets` returns buckets from every
region, and a client can reach a bucket homed elsewhere without the redirect AWS
gives it. This fork keys buckets by `(account, region, name)`.

What changes for a client:

- `ListBuckets` answers with the signing region's buckets only.
- A bucket lives in exactly one region. `CreateBucket` with a `LocationConstraint`
  other than the signing region still creates it where the constraint says, and
  later mutations of that record keep addressing it there.
- A request signed for the wrong region gets `301 PermanentRedirect` with
  `x-amz-bucket-region` naming the bucket's home — SDKs follow it, a plain client
  does not.
- A bucket **name** stays unique partition-wide: the same name in another region,
  or another account, is never a second bucket.
- **Creating a bucket you already own follows AWS.** In the bucket's own region a
  re-create answers `200` in us-east-1 and `409 BucketAlreadyOwnedByYou`
  elsewhere — the same codes, messages and ACL effect as AWS, so a Terraform
  re-apply behaves as it does on AWS. The `200` is AWS's documented legacy
  wording — *"re-creating a bucket that you already own in the North Virginia
  Region … returns 200 OK and resets the bucket access control lists (ACLs)"* —
  not a `touch`: AWS does not treat the call as a no-op. The reset is ACLs only;
  objects, bucket policy, tags and versioning stay exactly as they were.
- A name owned by **another account**, or asked for from a region **other than the
  bucket's home**, is `409 BucketAlreadyExists` with one message,
  `Bucket <name> already exists`. AWS answers the owner `BucketAlreadyOwnedByYou`
  there (or the legacy `200` from us-east-1); the fork keeps one answer so a name
  cannot silently move between regions.
- Persisted S3 state moves to on-disk format v3; a v2 snapshot migrates by the
  region each bucket already records, so `PERSIST_STATE=1` upgrades in place.
- Objects that load before their bucket's metadata no longer leave a second,
  object-less bucket record behind: the importer's placeholder is moved onto the
  real region.

Scope: S3 only. No other service, tenancy or region rule changes in this fork.

### Verified

Checked on 2026-11-12 against a running emulator, path-style through a gateway on
`127.0.0.1:4571`, signed with a 12-digit account root key:

| Call | Result |
| --- | --- |
| create a bucket in us-east-1 | `200` |
| create it again, same owner and region | `200`; a `public-read` ACL is back to private |
| create it again in eu-west-1 | `BucketAlreadyOwnedByYou` (409) |
| create it again from another account | `BucketAlreadyExists` (409) |
| `ListBuckets` in us-east-1 vs eu-west-1 | each region lists only its own bucket |

The first two rows live in `tests/test_s3.py`
(`test_s3_create_bucket_owned_again_resets_the_acl`,
`test_s3_create_owned_bucket_again`) and run with the rest of the S3 suite; the
full file passed with 1,160 tests.

## Multi-tenancy and regions

Nothing about the tenancy model changes here: an **account** is a 12-digit string,
and a client that signs with a 12-digit access key *is* that account for the
length of the request. The **region** comes from the SigV4 credential scope, so
two clients pointed at different regions see independent state. S3 is now the
last regional service to join that rule; Aurora DSQL is the only exception.

`Organizations CreateAccount` records a new 12-digit id in the organization and
that id becomes a usable tenancy the same way — sign with it. Every account also
has a **root** identity: the 12-digit key itself, accepted with any secret. Run
with `AUTH=true` and SigV4 signature checking applies to IAM access keys, not to
roots.

## Accounts, logins and access keys

Real credentials — a login and a secret access key — are IAM objects inside the
account:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566

# 1. Take an account id: any 12 digits, or the id Organizations CreateAccount returned.
export AWS_ACCESS_KEY_ID=111111111111
export AWS_SECRET_ACCESS_KEY=root          # root keys are not signature-checked

# 2. A login: the IAM user and its console password
aws iam create-user --user-name alice
aws iam create-login-profile --user-name alice \
  --password 'Correct-Horse-1' --password-reset-required

# 3. Let it do things, then issue a secret access key (shown once)
aws iam put-user-policy --user-name alice --policy-name admin \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}'
aws iam create-access-key --user-name alice
# → { "AccessKey": { "AccessKeyId": "AKIA…", "SecretAccessKey": "…", … } }
```

The **login profile** is stored and readable (`GetLoginProfile`,
`UpdateLoginProfile`, `DeleteLoginProfile`), but MiniStack serves no sign-in page
— it documents the credential for your own console or test harness, and the
**access key** is what clients sign with. The user's ARN is
`arn:aws:iam::<account>:user/<name>`, and with `AUTH=true` the pair is verified
the way AWS verifies it: an unknown key, a wrong secret or a replayed clock all
fail.

An **account ID is a tenancy, not a user**. One account holds any number of IAM
users — each with its own login profile and access keys — plus roles and groups;
a new account starts with none. In `arn:aws:iam::111111111111:user/alice`,
`111111111111` is the tenancy and `alice` is one identity inside it; adding `bob`
does not create a second account, and deleting every IAM user leaves the account
(and its root identity) in place.

```bash
# Sign as the IAM user from here on
export AWS_ACCESS_KEY_ID=AKIA…
export AWS_SECRET_ACCESS_KEY=…
export AWS_DEFAULT_REGION=us-east-1
aws sts get-caller-identity
# → { "Account": "111111111111", "Arn": "arn:aws:iam::111111111111:user/alice", ... }
```

## Using a service, e.g. S3

A bucket is account- and region-scoped, so the same commands answer differently
per region — and `us-east-1` is the one region where `mb` sends no
`LocationConstraint`:

```bash
aws s3 mb s3://alice-artifacts                       # us-east-1
aws s3 cp ./file.txt s3://alice-artifacts/

aws s3api create-bucket --bucket alice-eu --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1

aws s3 ls --region us-east-1                         # alice-artifacts
aws s3 ls --region eu-west-1                         # alice-eu
```

Two rules clients see, because they are S3's own:

- A request signed for the wrong region is `301 PermanentRedirect` with
  `x-amz-bucket-region` naming the bucket's home region — SDKs follow it, a
  plain client does not.
- A name is unique partition-wide: another account, or another region of this
  account, creating `alice-artifacts` gets `409 BucketAlreadyExists` with one
  message and nothing about who owns it.

## Building and tagging

The fork's only other commit is `.github/workflows/fork-image.yml`, which builds
`Dockerfile.full` and pushes
`ghcr.io/mathewmusango/ministack-upstream` on workflow dispatch and on `v*` tags
— never on a `main` push, because `main` carries no patch. Cutting a release:
merge `feat/s3-region-scoped-state` into `main`, tag `main`
(`v<upstream>-fork.<n>`), push the tag. A tag-triggered build reads the workflow
from the tagged commit, which is why it lives on `main`.

**A tag runs the published build, not the checkout.** An image reference names
what was pushed; a change in the working tree takes effect only through a build
from this checkout — `podman build -f Dockerfile.full -t localhost/ministack:fork-full .`
(the lab's `MINISTACK_BUILD=1`) — or a new published tag. Running
`1.5.40-fork.1-full` runs that release, without any working-tree change.

### Tags

Upstream publishes two variants from one release: the Alpine image as
`<version>`, `<major>.<minor>` and `latest`, and the Debian/full image as
`<version>-full`, `<major>.<minor>-full` and a floating `full`. This fork builds
**only the full variant** (`Dockerfile.full`), so it publishes:

| Tag | Meaning | Moves? |
| --- | --- | --- |
| `1.5.40-fork.1-full` | the release, what `MINISTACK_IMAGE` pins | no |
| `v1.5.40-fork.1-full` | the git tag name, same image | no |
| `sha-<short>-full` | every build, for "what exactly is running?" | no |
| `feat-s3-region-scoped-state-full` | the branch head | yes |

Not published: `latest`, `1.5.40` and `1.5` name the **slim** image, which this
fork does not build; `full` and `1.5-full` are upstream's floating aliases and
have no fork equivalent yet. Pin the exact `-full` tag in anything reproducible.

This file is maintained by hand alongside the branch. It is **not** part of the
upstream pull request.
