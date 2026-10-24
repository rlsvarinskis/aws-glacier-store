# aws-glacier-store

Encrypted, metadata-preserving archives of large files in Amazon S3 Glacier, restored byte for byte.

- Every file gets its own random 256-bit key. Keys never leave your machine.
- A local SQLite database records each upload: its key, the original SHA-512, size, owner, permissions and timestamps.
- Uploads stream straight from disk into S3, with no temporary copy.
- Downloads go through a private CloudFront distribution, which is free up to 1 TB a month.
- The bucket stays private; only URLs signed with your own key can read from it.

```bash
glacier-put ~/backups/laptop.img.xz disk-images/        # archive (Deep Archive by default)
glacier-get disk-images/laptop.img.xz /mnt/restore      # restore and verify
glacier-db list                                         # what is archived
```

## How it works

```
upload:    file ──► tar ──► gpg encrypt ──► aws s3 cp ──► S3 bucket
                     │       ▲
           checksums │       │ new key
                     ▼       │
                glacier.db: keys, SHA-512, size, owner, timestamps
                     │       │
       SHA-512 check │       │ key
                     ▼       ▼
download:  file ◄── tar ◄── gpg decrypt ◄── CloudFront ◄── S3 bucket
                                          (signed URL)
```

- **One pass, no temporary copy.** The stages are joined by pipes, so the chain runs at the speed of the upload. The AWS
  CLI buffers up to about 640 MB of parts in memory; everything else uses a few MB.
- **Key first.** A new key is generated for every upload and written to the database before any data leaves, so a
  finished upload can never lack its key. It reaches gpg through a pipe and is never written anywhere else.
- **Checks.** The SHA-512 of the original file and the SHA-256 of the encrypted object are computed while streaming.
  A restore checks the download's SHA-256 before decrypting and the restored file's SHA-512 afterwards; gpg's own
  integrity check also rejects any altered byte.
- **Standard formats.** A POSIX tar of the single file (owner, permissions, nanosecond timestamps, extended attributes,
  ACLs), encrypted with `gpg --symmetric` (AES-256, compression off). Any machine with tar and gpg can restore it by
  hand.

## Requirements

AWS CLI v2, GnuPG 2, GNU tar, pv, Python 3, OpenSSL, curl, and sqlite3 for recovery by hand.

```bash
# Arch Linux
pacman -S aws-cli-v2 gnupg tar pv python openssl curl sqlite
```

## Setup

Replace `BUCKET`, `REGION` and `USER` with your own names. Steps 3–8 need an administrator login, either in the AWS
console or as a separate CLI profile (e.g. prefix the commands with `AWS_PROFILE=admin`); the tools themselves run
with the IAM user's access key from step 6.

1. **Install the tools** into one directory on your PATH. They must sit together, because `glacier-put` and
   `glacier-get` call `glacier-db` from their own directory.
   ```bash
   install -m 755 bin/* ~/.local/bin/
   ```
2. **Write the config file** `$XDG_CONFIG_HOME/glacier/config` (default `~/.config/glacier/config`), as in
   [`config.example`](config.example):
   ```bash
   GLACIER_BUCKET=BUCKET
   GLACIER_REGION=REGION
   ```
3. **Create the bucket** in the S3 console, or with the CLI. New buckets already block public access, disable ACLs and
   encrypt by default; keep those settings.
   ```bash
   aws s3api create-bucket --bucket BUCKET --region REGION --create-bucket-configuration LocationConstraint=REGION
   ```
4. **Add a lifecycle rule** that clears interrupted uploads after 7 days (console: bucket → Management → Create
   lifecycle rule → "Delete expired object delete markers or incomplete multipart uploads").
   ```bash
   aws s3api put-bucket-lifecycle-configuration --bucket BUCKET --lifecycle-configuration \
     '{"Rules":[{"ID":"abort-incomplete-uploads","Status":"Enabled","Filter":{},"AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}}]}'
   ```
5. **Create the IAM user** the tools run as, without console access, and give it the bucket access policy
   [`policies/glacier-bucket-access.json`](policies/glacier-bucket-access.json) as an inline policy. It allows upload,
   read and restore on this bucket only, and deliberately no `s3:DeleteObject`: a compromised machine can't wipe the
   archive.
   ```bash
   sed s/BUCKET/your-bucket/g policies/glacier-bucket-access.json > /tmp/glacier-bucket-access.json
   aws iam create-user --user-name USER
   aws iam put-user-policy --user-name USER --policy-name glacier-bucket-access \
     --policy-document file:///tmp/glacier-bucket-access.json
   ```
6. **Create an access key** for that user (console: the user's Security credentials → Create access key → Command Line
   Interface, or `aws iam create-access-key --user-name USER`) and enter it on the machine with `aws configure`.
7. **Set up the download path once.** Give the user the temporary policy
   [`policies/glacier-cdn-setup.json`](policies/glacier-cdn-setup.json) as a second inline policy, then run
   `glacier-cdn-setup` with the user's own key:
   ```bash
   sed s/BUCKET/your-bucket/g policies/glacier-cdn-setup.json > /tmp/glacier-cdn-setup.json
   aws iam put-user-policy --user-name USER --policy-name glacier-cdn-setup \
     --policy-document file:///tmp/glacier-cdn-setup.json
   glacier-cdn-setup
   ```
   It creates a signing key (kept locally), a CloudFront public key and key group, an origin access control, a
   distribution (HTTPS only, caching off, signed URLs only) and a bucket policy that lets only that distribution read.
   Wait until the distribution shows *Deployed* in the CloudFront console (a few minutes).
8. **Remove the temporary policy.** Downloads need no CloudFront permissions; URLs are signed offline.
   ```bash
   aws iam delete-user-policy --user-name USER --policy-name glacier-cdn-setup
   ```
9. **Test** with a dry run, then a small upload in Standard and a restore. The upload user can't delete, so remove the
   test object in the S3 console afterwards.
   ```bash
   glacier-put --dry-run ~/notes.txt test/
   glacier-put --class STANDARD ~/notes.txt test/
   glacier-get test/notes.txt /tmp/restore-test
   ```

## Usage

Run long uploads inside `tmux` or `screen`, so a closed terminal doesn't kill them. Every command prints full help with
`--help`.

| Command | Option | What it does |
| --- | --- | --- |
| `glacier-put FILE [NAME]` | `--class CLASS` | `DEEP_ARCHIVE` (default), `GLACIER`, `GLACIER_IR`, `STANDARD_IA` or `STANDARD` |
| | `--bandwidth RATE` | Caps the upload speed, e.g. `20MB/s` |
| | `--dry-run` | Does everything except the upload; recorded as a dry run |
| `glacier-get NAME [DEST_DIR]` | `--stream` | Restores while downloading: no room needed for a second copy, but no resume |
| | `--tier Bulk` or `Standard` | Speed of the restore inside AWS for archived files (default Bulk) |
| | `--days N` | How long AWS keeps the restored copy readable (default 3) |
| | `--passphrase-file FILE` | For objects not in the database; NAME is then the exact S3 key |
| `glacier-db` | `list [--all]` | Lists uploads; `--all` adds dry runs and failed attempts |
| | `show NAME` | Everything recorded for one upload, except its key |
| | `show-key NAME` | Prints the upload's encryption key |
| `glacier-cdn-setup` | | One-time setup of the CloudFront download path |

`glacier-put` refuses a name that is already in the archive. `glacier-get` restores into the current directory when
`DEST_DIR` is left out, never overwrites an existing file, and resumes an interrupted download when run again. For a
file in Glacier or Deep Archive, its first run asks before starting a (paid) restore inside AWS and exits; run the same
command again once the restore has finished.

### Names

A file's archive name is its full path in the bucket, folder included, and it is always referred to by that exact name.

| You run | Archive name | Restored as |
| --- | --- | --- |
| `glacier-put ~/backups/laptop.img.xz` | `laptop.img.xz` | `laptop.img.xz` |
| `glacier-put ~/backups/laptop.img.xz disk-images/` | `disk-images/laptop.img.xz` | `laptop.img.xz` |
| `glacier-put ~/backups/laptop.img.xz disk-images/old-pc.xz` | `disk-images/old-pc.xz` | `old-pc.xz` |

A name without its folder is rejected with a "did you mean" hint.

## Storage classes and costs

Deep Archive is the default: about 23 times cheaper than Standard, at the price of a 180-day minimum and up to 48 hours
to restore. Example list prices for eu-north-1 (Stockholm), October 2026; check
[S3 pricing](https://aws.amazon.com/s3/pricing/) for your region.

| `--class` | Storage per GB-month | 1 TiB per month | Minimum billed | Restore wait | Restore fee per GB |
| --- | --- | --- | --- | --- | --- |
| `DEEP_ARCHIVE` | ≈ $0.00099 | ≈ $1.01 | 180 days | ≤ 12 h (Standard), ≤ 48 h (Bulk) | $0.02 (Standard), $0.003 (Bulk) |
| `GLACIER` | $0.0036 | $3.69 | 90 days | 3–5 h (Standard), 5–12 h (Bulk) | $0.01 (Standard), free (Bulk) |
| `GLACIER_IR` | $0.004 | $4.10 | 90 days | none | $0.03 |
| `STANDARD_IA` | $0.0125 | $12.80 | 30 days | none | $0.01 |
| `STANDARD` | $0.023 | $23.55 | none | none | none |

- A restored copy is billed at the Standard rate for the days it is kept (`--days`).
- Downloads through CloudFront are free up to 1 TB and 10 million requests a month, and S3 → CloudFront transfer is
  free ([CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/pay-as-you-go/)). Downloading straight from S3
  pays normal S3 transfer fees.
- To change the class later (e.g. upload in Standard, test a restore, then archive), copy the object onto itself; the
  console's "Edit storage class" only works below 5 GB:
  ```bash
  aws s3 cp s3://BUCKET/NAME s3://BUCKET/NAME --storage-class DEEP_ARCHIVE --copy-props metadata-directive --region REGION
  ```
  The minimum billed time starts at the conversion. `glacier-get` always checks the live class in S3.

## Files and directories

The tools follow the XDG base directory spec.

| Path | Contents |
| --- | --- |
| `$XDG_CONFIG_HOME/glacier/config` (default `~/.config/glacier/config`) | `GLACIER_BUCKET`, `GLACIER_REGION` |
| `$XDG_DATA_HOME/glacier/glacier.db` (default `~/.local/share/glacier`) | Per-file keys, checksums, sizes, owners, timestamps, upload status |
| `$XDG_DATA_HOME/glacier/cloudfront-signing-key.pem` | Private key that signs download URLs |
| `$XDG_DATA_HOME/glacier/cdn.env` | CloudFront domain, distribution and key IDs |

Environment variables `GLACIER_BUCKET`, `GLACIER_REGION`, `GLACIER_CONFIG` and `GLACIER_STATE_DIR` override the
defaults. Every shell that runs the tools must see the same `XDG_DATA_HOME`: if it is only set in a login profile, a
command run as `ssh host glacier-get …` looks in `~/.local/share/glacier`, finds no database and stops.

## Backup and disaster recovery

**Back up `$XDG_DATA_HOME/glacier` off the machine after every upload.** `glacier.db` holds the only copies of the keys;
without it nothing in the bucket can ever be decrypted. Keep the backup encrypted.

The format is plain tar inside standard gpg encryption, so a file can be restored by hand with only the database backup
and the AWS CLI:

```bash
# 1. The file's key
sqlite3 glacier.db "SELECT passphrase FROM uploads WHERE s3_key='disk-images/laptop.img.xz' AND status='uploaded' ORDER BY started_at DESC LIMIT 1" > key.txt
# 2. If it is in Glacier or Deep Archive: restore it, then wait until head-object shows ongoing-request="false"
aws s3api restore-object --region REGION --bucket BUCKET --key disk-images/laptop.img.xz \
  --restore-request '{"Days":3,"GlacierJobParameters":{"Tier":"Bulk"}}'
# 3. Download (normal S3 transfer fees; `aws cloudfront sign` with the backed-up signing key makes a free URL instead)
aws s3 cp --region REGION s3://BUCKET/disk-images/laptop.img.xz laptop.img.xz.encrypted
# 4. Decrypt and unpack (as root to restore the original owner)
gpg --batch --pinentry-mode loopback --passphrase-file key.txt --decrypt laptop.img.xz.encrypted | tar -xpf - --xattrs --acls
# 5. Compare `sha512sum laptop.img.xz` with source_sha512 in the same database row, then delete key.txt
```

## Failures and cleanup

A failed upload never leaves a half-written file in the bucket: S3 only creates the object once the last part is
confirmed. Retrying is always safe; each attempt gets a new database row and a new key, and only successful uploads
are ever used.

| What happened | What is left over | What to do |
| --- | --- | --- |
| Ctrl+C, network gives up, an error | Database row marked `failed`; AWS discards the uploaded parts | Run the same command again |
| Hard kill (power loss, `kill -9`, terminal closed outside tmux) | Row stays `uploading`; uploaded parts stay in S3, invisible but billed | The lifecycle rule from setup step 4 removes them after 7 days, or cancel them now (below) |

```bash
aws s3api list-multipart-uploads --bucket BUCKET --region REGION
aws s3api abort-multipart-upload --bucket BUCKET --region REGION --key KEY --upload-id UPLOAD_ID
```

## Known limitations

- If an upload finishes but the final size check fails (e.g. a brief network drop at the very end), the row is marked
  `failed` although the file is in S3 with its key in the database. `glacier-get` then won't use it and `glacier-put`
  refuses the name. A `glacier-db repair` command is planned.
- Downloading a restored Glacier or Deep Archive object through CloudFront has not been verified end to end yet; the
  direct S3 route in the recovery steps always works.
- Creation (birth) time is not preserved: tar has no field for it. Change time (ctime) is recorded but can't be set on
  restore by any tool.
- An interrupted upload restarts from the beginning; there is no local copy to resume from.
- One regular file per upload. Archive a directory by making a tar of it first.

## License

[MIT](LICENSE)
