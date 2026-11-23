# CloudFront Edge Location Prewarming

When an object is created or overwritten under a configured S3 path, a Lambda function downloads it once from every CloudFront edge location in the configured countries or cities, so those locations cache the new version before users ask for it.

> **Disclaimer.** This is sample code, for demonstration and learning. It is not intended for production use without your own security, cost and operational review and testing.

![Architecture diagram](architecture.png)

```
S3 bucket (any region) ──> default event bus ──forwarding rule──> cdn-prewarm-bus (central region) ──> prewarm Lambda
                                                                                      1. invalidate the URL
                                                                                      2. GET it from each edge location
```

## Deployment

The buckets, the CloudFront distributions, and the central stack must be in the same AWS account.

**1. Deploy the central stack** once, in any region:

```bash
aws cloudformation deploy --region us-east-1 --stack-name cdn-prewarm \
  --template-file dist/cdn-prewarm-central.yaml --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    "S3Paths=bucket-a/static/;bucket-b/img/" \
    "CdnUrlPrefixes=https://www.example.com/static/;https://img.example.com/" \
    "EdgeCountriesOrCities=Tokyo;Germany;United States" \
    "FilePatterns=*"
```

**2. Deploy the forwarder stack** in every region that has a bucket, using the central stack's `EventBusArn` output:

```bash
aws cloudformation deploy --region us-west-2 --stack-name cdn-prewarm-forwarder \
  --template-file dist/cdn-prewarm-forwarder.yaml --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    "CentralEventBusArn=arn:aws:events:us-east-1:111122223333:event-bus/cdn-prewarm-bus" \
    "BucketNames=bucket-a" \
    "KeyPrefixes=static/;img/"
```

`deployer-policy.json` has the IAM permissions needed to deploy. It only covers resources named `cdn-prewarm*`, so keep that prefix in stack names. Deploying from the console also needs `s3:CreateBucket`, `s3:PutObject`, and `s3:GetObject` on `cf-templates-*` buckets.

**`deployer-policy.json` is admin-equivalent in the account. Give it only to trusted deployers.** It lets the deployer create `cdn-prewarm*` roles with any inline policy, pass them to Lambda, and run any code in `cdn-prewarm*` functions, so the deployer can act with any permission. To limit that, add a [permissions boundary](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html): require it with an `iam:PermissionsBoundary` condition on `iam:CreateRole` and `iam:PutRolePolicy`, deny `iam:DeleteRolePermissionsBoundary`, and set `PermissionsBoundary` on the roles in both templates. ARNs in the policy use `arn:*:` so it works in every AWS partition.

## Parameters

List parameters are separated by `;`.

**Central stack**

| Parameter | Description |
|---|---|
| S3Paths | Paths to monitor, as `bucket/prefix/` or just `bucket`. |
| CdnUrlPrefixes | One URL prefix per S3 path, in the same order. |
| EdgeCountriesOrCities | Countries and/or cities, for example `Tokyo;Germany;United States` (the default). |
| FilePatterns | File name patterns to prewarm, for example `*.mp4;*.m3u8`. Default `*`. |
| InvalidateBeforePrewarm | Default `true`. Keep it on if files get overwritten. |
| ReservedConcurrency | Maximum number of files prewarmed at the same time. Default `10`; empty removes the cap. See [Cost and limits](#cost-and-limits). |
| EdgeLocationsUrl | Optional. Empty (default): use only the edge location snapshot bundled in the code, nothing is downloaded. See [Edge location list](#edge-location-list). |

The prewarm URL is the URL prefix plus the rest of the key: with `bucket-a/static/` → `https://www.example.com/static/`, uploading `static/js/app.js` prewarms `https://www.example.com/static/js/app.js`. The Lambda finds the distribution by the URL's domain.

Country and city names are those in the edge location list (by default the `EDGE_SNAPSHOT` block in `src/prewarm.py`). Matching ignores case, accents, and punctuation, and the state code can be omitted (`Los Angeles` matches `Los Angeles, CA`). Hong Kong (`Hong Kong`) and Taiwan (`Taiwan`, city `Taipei`) are listed as their own places. To print every country and city:

```bash
python3 -c "import ast,re; s=open('src/prewarm.py').read(); d=ast.literal_eval(re.search(r'^EDGE_SNAPSHOT = (\{.*?^\})', s, re.S|re.M)[1]); [print(c, '-', '; '.join(d[c])) for c in d]"
```

**Forwarder stack**

| Parameter | Description |
|---|---|
| CentralEventBusArn | `EventBusArn` output of the central stack. |
| BucketNames | Buckets in this region, for example `bucket-a;bucket-b`. |
| KeyPrefixes | Optional. Only forward objects under these prefixes. |
| EnableBucketEventBridge | Default `true`: turns on EventBridge notifications for the buckets. |

## Edge location list

The Lambda needs to know which edge locations exist in each country and city. By default it uses the snapshot bundled in `src/prewarm.py` and makes no request for it at run time. The snapshot date is in `EDGE_SNAPSHOT_DATE`; edge locations change over time, so refresh it now and then (see [Development](#development)).

AWS does not publish a list of edge location codes. The bundled snapshot was built from [static.cloudping.cloud/cdn/cloudfront-edge-locations.json](https://static.cloudping.cloud/cdn/cloudfront-edge-locations.json), **a third-party list that is not maintained or endorsed by AWS**; check its accuracy and terms of use before you rely on it. To have the Lambda download a list at run time instead, set `EdgeLocationsUrl` to an https URL that you trust. The list must have this shape:

```json
{"edge_locations": [{"country": "Japan", "city": "Tokyo", "active_nodes": [{"code": "NRT12-P5"}]}]}
```

The list is cached for an hour, and the snapshot is used when the download fails. A list can only choose which edge locations are requested: codes must match `AAA00-XX`, and requests always go to `<distribution>.<code>.cloudfront.net`.

## Viewing results

Logs are in `/aws/lambda/<central stack name>-prewarm`, one `prewarm_result` per edge location and one `prewarm_summary` per file. Locations that did not succeed:

```
fields @timestamp, url, node, outcome, error
| filter type = "prewarm_result" and ok = 0
| sort @timestamp desc
```

`outcome` is `ok`, `failed`, or, when the Lambda ran out of time, `not_started` / `aborted`.

## Verifying with curl

```bash
DOMAIN=www.example.com              # domain in CdnUrlPrefixes
FILE_PATH='/static/js/my_app.js'
NODE=NRT12-P5                       # location code from this file's prewarm_result logs
DIST=d111111abcdef8                 # prefix of the distribution domain dxxxx.cloudfront.net

URL_PATH=$(jq -rn --arg p "$FILE_PATH" '$p | split("/") | map(@uri) | join("/")')

curl -sI -m 15 --retry 3 --retry-all-errors -H 'Accept-Encoding: gzip, deflate, br' \
  --connect-to "$DOMAIN:443:$DIST.$NODE.cloudfront.net:443" \
  "https://$DOMAIN$URL_PATH" | grep -iE '^(HTTP/|x-cache:|x-amz-cf-pop:|etag:|age:)'
```

A successful prewarm shows `x-amz-cf-pop` equal to `NODE`, `x-cache: Hit from cloudfront` (or `RefreshHit`, meaning the cached copy expired and was revalidated with the origin), and the same `etag` as in S3. No output means your network can't reach that location.

## Cost and limits

Every created or overwritten file that matches a route costs:

- **Data transfer out of CloudFront**: about `file size × number of edge locations`, billed at the CloudFront rate of each location's region. The default `Tokyo;Germany;United States` selects 81 locations, so a 100 MB file transfers about 8 GB. Each location also counts one HTTPS request.
- **One invalidation path** when `InvalidateBeforePrewarm` is `true`. The first 1,000 paths per month are free across the account, then each path is charged (USD 0.005 per path at the time of writing). For example, 10,000 uploads in a month cost about USD 45 in invalidations. See [Paying for file invalidation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PayingForInvalidation.html) and [CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/).
- **Lambda duration**: 1,024 MB for the invalidation wait (often 10 to 60 seconds, at most 300) plus the downloads.

A bulk upload of N files starts N prewarms, each downloading the file from every selected location. `ReservedConcurrency` (default 10) caps how many run at once. The rest wait in the Lambda asynchronous queue and are dropped after 6 hours, so at about one minute per file, roughly 3,600 files can be prewarmed per burst. Raise the cap for larger bursts, keeping in mind that CloudFront accepts at most 150 invalidation paths per second ([quotas](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html#limits-invalidations)) and that reserved concurrency is taken from the account's Lambda concurrency, which must keep at least 100 unreserved. Set budgets and alarms (for example AWS Budgets) before enabling this on busy paths.

## Caveats

- **Caching must be enabled** on the distribution, otherwise prewarming does nothing.
- **Per-location host names are not a documented CloudFront interface.** The Lambda reaches each location through `<distribution>.<code>.cloudfront.net` (for example `d111111abcdef8.NRT12-P5.cloudfront.net`). CloudFront does not document these names, and they may stop working or change without notice.
- **Don't monitor paths that logs are written to**, or each prewarm triggers another.
- **Failures.** A failed location is not retried. If no location succeeds, Lambda retries the whole event twice, then drops it. To prewarm again, upload the file again.
- **Permissions.** The Lambda may invalidate any distribution in the account (`distribution/*`), because it finds distributions at run time from the `CdnUrlPrefixes` host names.

## Cleanup

1. Delete the forwarder stack in every region where you deployed it, then the central stack:

   ```bash
   aws cloudformation delete-stack --region us-west-2 --stack-name cdn-prewarm-forwarder
   aws cloudformation delete-stack --region us-east-1 --stack-name cdn-prewarm
   ```

   This removes the rules, the event bus, the Lambda functions, their roles and their log groups (including the logs).
2. **EventBridge notifications stay enabled on the buckets.** The forwarder stack turns them on but leaves them on when deleted, because other applications may use them. With no rules left, the events are not delivered anywhere. To turn them off, open the bucket in the S3 console, choose **Properties**, and in **Amazon EventBridge** choose **Off**. Or, keeping the bucket's other notification settings:

   ```bash
   B=bucket-a
   aws s3api get-bucket-notification-configuration --bucket "$B" \
     | jq 'del(.EventBridgeConfiguration)' > notification.json
   aws s3api put-bucket-notification-configuration --bucket "$B" \
     --notification-configuration file://notification.json
   ```

## Development

After editing `src/`, run `python3 build.py` to regenerate `dist/`. To refresh the bundled edge location snapshot from a list of the shape above, run `python3 refresh_edge_snapshot.py <edge-list-url>`, then `python3 build.py`.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
