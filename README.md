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

Now let's set a retention policy on the bucket for 3 days:

```bash
> gsutil retention set 3d gs://${BUCKET}
Setting Retention Policy on gs://<YOUR_PROJECT>-pol-test/...
> gsutil retention get gs://${BUCKET}
  Retention Policy (UNLOCKED):
    Duration: 3 Day(s)
    Effective Time: Tue, 15 Jun 2021 17:06:05 GMT
```

Now let's set the retention policy lock it:

```bash
> gsutil retention lock gs://${BUCKET}
This will PERMANENTLY set the Retention Policy on gs://<YOUR_PROJECT>-pol-test to:

  Retention Policy (UNLOCKED):
    Duration: 3 Day(s)
    Effective Time: Tue, 15 Jun 2021 17:06:05 GMT

This setting cannot be reverted!  Continue? [y|N]: y
Locking Retention Policy on gs://<YOUR_PROJECT>-pol-test/...

> gsutil retention get gs://${BUCKET}
  Retention Policy (LOCKED):
    Duration: 3 Day(s)
    Effective Time: Tue, 15 Jun 2021 17:06:05 GMT
```

Also as a quick test I was able to confirm I am not able to delete our test file:

```bash
> gsutil rm gs://${BUCKET}/test.txt
Removing gs://<YOUR_PROJECT>-pol-test/test.txt...
AccessDeniedException: 403 Object '<YOUR_PROJECT>-pol-test/test.txt' is subject to bucket's retention policy and cannot be deleted, overwritten or archived until 2021-06-18T09:52:05.796309-07:00
```

## Confirming Lifecycle Management Changed the Class of a File in a Locked Bucket

After 24 hours I checked the file and the class had not changed. I looked over our documentation ([Object lifecycle behavior](https://cloud.google.com/storage/docs/lifecycle#behavior)) we have a note that it could take 24 hours to apply the lifecycle change:

> Note: Updates to your lifecycle configuration may take up to 24 hours to go into effect. This means that when you change your lifecycle configuration, Object Lifecycle Management may still perform actions based on the old configuration for up to 24 hours.

Because my rule was to change files that are 1 day old to be switch to **Nearline**, but since it takes an additional 24 hours to make the change I could have to wait up to approximately 48 hours. I checked 42 hours later and the change was applied:

```bash
> gsutil stat gs://${BUCKET}/test.txt
gs://<YOUR_PROJECT>-pol-test/test.txt:
    Creation time:          Tue, 15 Jun 2021 16:52:05 GMT
    Update time:            Thu, 17 Jun 2021 10:36:29 GMT
    Storage class update time:Thu, 17 Jun 2021 10:36:29 GMT
    Storage class:          NEARLINE
    Retention Expiration:   Fri, 18 Jun 2021 16:52:05 GMT
    Content-Language:       en
    Content-Length:         20
    Content-Type:           text/plain
    Hash (crc32c):          XXXX==
    Hash (md5):             XXX==
    ETag:                   CI7qXXX=
    Generation:             1623775925728526
    Metageneration:         2
```

You can see that the file was created `June 15 16:52` and  the class was changed `June 17 10:36`, so it took `1 day, 17 hours` (**41.7** hours)... It took longer than 24 hours to apply the change which is expected as per the above note. But we can also see that the retention lock is valid until `June 18 16:52`, which is within the time frame when the object class change occured. This confirms that [Object Lifecycle Management](https://cloud.google.com/storage/docs/lifecycle) works with [Retention policies and retention policy locks](https://cloud.google.com/storage/docs/bucket-lock). Just to be 100% sure I tried to delete the file and it failed:

```bash
> gsutil rm gs://${BUCKET}/test.txt
Removing gs://<YOUR_PROJECT>-pol-test/test.txt...
AccessDeniedException: 403 Object '<YOUR_PROJECT>-pol-test/test.txt' is subject to bucket's retention policy and cannot be deleted, overwritten or archived until 2021-06-18T09:52:05.796309-07:00
```

