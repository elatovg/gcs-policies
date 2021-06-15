# gcs-policies

Quick test to make sure [Object Lifecycle Management](https://cloud.google.com/storage/docs/lifecycle) work with [Retention policies and retention policy locks](https://cloud.google.com/storage/docs/bucket-lock)

## Create the GCS Bucket
Here is an easy way to test it out in your project:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export BUCKET="${PROJECT_ID}-pol-test"
gsutil mb gs://${BUCKET}
```

Next let's copy a test file:

```bash
gsutil cp test.txt gs://${BUCKET}
```

Now let's confirm the **class** of the test file:

```bash
> gsutil stat gs://${BUCKET}/test.txt
gs://<YOUR_PROJECT>-pol-test/test.txt:
    Creation time:          Tue, 15 Jun 2021 16:52:05 GMT
    Update time:            Tue, 15 Jun 2021 16:52:05 GMT
    Storage class:          STANDARD
    Content-Language:       en
    Content-Length:         20
    Content-Type:           text/plain
    Hash (crc32c):          XXX==
    Hash (md5):             XX==
    ETag:                   XXX=
    Generation:             1623775925728526
    Metageneration:         1
```

## Create the  Object Lifecycle Management Policy 
Now let's create the  [Object Lifecycle Management](https://cloud.google.com/storage/docs/lifecycle) policy. Here is a sample policy to change from standard to nearline after one day:

```bash
> cat lc.json
{
    "lifecycle": {
      "rule": [
      {
        "action": {
          "type": "SetStorageClass",
          "storageClass": "NEARLINE"
        },
        "condition": {
          "age": 1,
          "matchesStorageClass": ["MULTI_REGIONAL", "STANDARD", "DURABLE_REDUCED_AVAILABILITY"]
        }
      }
      ]
    }
}%
```

By default there is no policy:

```bash
> gsutil lifecycle get gs://${BUCKET}
gs://<YOUR_PROJECT>-pol-test/ has no lifecycle configuration.
```

Now let's set the policy:

```bash
> gsutil lifecycle set lc.json gs://${BUCKET}
Setting lifecycle configuration on gs://<YOUR_PROJECT>-pol-test/...
```

Confirm the policy is there:

```bash
> gsutil lifecycle get gs://${BUCKET}
{"rule": [{"action": {"storageClass": "NEARLINE", "type": "SetStorageClass"}, "condition": {"age": 1, "matchesStorageClass": ["MULTI_REGIONAL", "STANDARD", "DURABLE_REDUCED_AVAILABILITY"]}}]}
```

## Configuring the Retention policies and retention policy locks

Now let's set a retention policy on the bucket for 2 days:

```bash
> gsutil retention set 2d gs://${BUCKET}
Setting Retention Policy on gs://<YOUR_PROJECT>-pol-test/...
> gsutil retention get gs://${BUCKET}
  Retention Policy (UNLOCKED):
    Duration: 2 Day(s)
    Effective Time: Tue, 15 Jun 2021 17:06:05 GMT
```

Now let's set the retention policy lock it:

```bash
> gsutil retention lock gs://${BUCKET}
This will PERMANENTLY set the Retention Policy on gs://<YOUR_PROJECT>-pol-test to:

  Retention Policy (UNLOCKED):
    Duration: 2 Day(s)
    Effective Time: Tue, 15 Jun 2021 17:06:05 GMT

This setting cannot be reverted!  Continue? [y|N]: y
Locking Retention Policy on gs://<YOUR_PROJECT>-pol-test/...

> gsutil retention get gs://${BUCKET}
  Retention Policy (LOCKED):
    Duration: 2 Day(s)
    Effective Time: Tue, 15 Jun 2021 17:06:05 GMT
```

Tomorrow I will check if the file has changed to nearline :). Also as a quick test I was able to confirm I am not able to delete our test file:

```bash
> gsutil rm gs://${BUCKET}/test.txt
Removing gs://<YOUR_PROJECT>-pol-test/test.txt...
AccessDeniedException: 403 Object '<YOUR_PROJECT>-pol-test/test.txt' is subject to bucket's retention policy and cannot be deleted, overwritten or archived until 2021-06-17T09:52:05.796309-07:00
```