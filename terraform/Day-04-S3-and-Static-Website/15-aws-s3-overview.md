# Day 04 — AWS S3 Overview

> S3 is AWS object storage: you create globally unique buckets and store any amount of data in them, manually via the console or as code with Terraform.

## Learning Objectives
- Explain what Amazon S3 is and when to use it.
- Apply S3 bucket naming rules correctly.
- Create a bucket manually in the AWS Console and upload an object.
- Understand why ACLs are disabled by default on new buckets.

## Prerequisites
- An AWS account with permission to use S3.
- A browser for the AWS Console.
- (For later lessons) AWS CLI configured and Terraform 1.x installed.

## Concept (plain English, beginner friendly)
S3 stands for **Simple Storage Service**. It is an **object storage** service — you store files ("objects") in containers called **buckets**, and retrieve them from anywhere over the internet.

A bucket is like a top-level folder that lives in one AWS region. Inside it you store objects identified by a **key**, e.g. `folder1/file1.txt`. S3 has no real folders; the slashes in a key just look like folders in the console (these are called **prefixes**).

Why people use S3:
- Backup & restore, disaster recovery, archiving
- Static website hosting
- Data lakes, logs & analytics
- Media storage, application data

Properties worth remembering: highly **durable**, **secure**, **scalable**, and **cost-effective**. Pricing is **pay as you go** — no minimum fees, no upfront cost. You pay for storage, requests, and data transfer.

**Bucket naming rules**
- Globally unique across all AWS accounts
- 3–63 characters
- Lowercase letters, numbers, dots (`.`) and hyphens (`-`) only
- Must start and end with a letter or number
- No spaces or special characters

| Name | Valid? | Why |
|---|---|---|
| `my-aws-bucket-12345` | Yes | lowercase, hyphens, unique-ish |
| `My AWS Bucket` | No | uppercase + spaces |
| `abc@123` | No | `@` not allowed |

**About ACLs:** on buckets created today, **Object Ownership = Bucket owner enforced**, which means ACLs are **disabled**. Access is controlled by IAM policies and bucket policies instead. That is why in modern Terraform you use `aws_s3_bucket_public_access_block` and `aws_s3_bucket_policy` rather than an `acl = "public-read"` setting.

## Step-by-Step Practical (Console)
1. Sign in to the AWS Console and search for **S3**.

```text
Console search bar -> "S3" -> S3 service
```

2. Click **Create bucket**.
3. Bucket type: **General purpose**.
4. Enter a **globally unique** bucket name. Add a random suffix so it does not clash:

```text
my-aws-bucket-12345
```

5. Choose an **AWS Region** (for example `ap-northeast-1`). Remember it — bucket names are global but buckets live in one region.
6. **Object Ownership**: leave **ACLs disabled (recommended)**.
7. **Block Public Access**: leave **all four settings ON** for now. (We only relax this in the static-website lesson.)
8. **Bucket Versioning**: optional — enable it if you want object history.
9. Click **Create bucket**.
10. Open the bucket, click **Upload**, add a small file (e.g. `myfile.txt` containing `Hello World`), then **Upload**.
11. Optionally click **Create folder** to make a prefix such as `folder1/` and upload into it.

Same thing with the AWS CLI, if you prefer the terminal:

```bash
# Create the bucket (region other than us-east-1 needs the location constraint)
aws s3api create-bucket \
  --bucket my-aws-bucket-12345 \
  --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1

# Upload a file
echo "Hello World" > myfile.txt
aws s3 cp myfile.txt s3://my-aws-bucket-12345/mydata.txt

# List objects
aws s3 ls s3://my-aws-bucket-12345/
```

## Expected Output
Console: the bucket appears in the **Buckets** list with its region and creation date, and the **Objects** tab shows your uploaded file.

CLI:

```text
make_bucket: my-aws-bucket-12345
upload: ./myfile.txt to s3://my-aws-bucket-12345/mydata.txt
2026-08-31 12:30:45         12 mydata.txt
```

## Verification
1. S3 Console -> **Buckets** -> your bucket exists in the region you chose.
2. Open the bucket -> **Objects** -> `mydata.txt` is listed with a size.
3. Click the object -> **Open** -> you see `Hello World`.
4. Copy the **Object URL** and open it in a private browser window — it should give **Access Denied**. That is correct: the bucket is private, and Block Public Access is on.

```bash
aws s3api head-object --bucket my-aws-bucket-12345 --key mydata.txt
```

## Cleanup
S3 charges for stored data, so remove practice buckets.

```bash
# Delete all objects, then the bucket
aws s3 rm s3://my-aws-bucket-12345/ --recursive
aws s3api delete-bucket --bucket my-aws-bucket-12345 --region ap-northeast-1
```

Console alternative: select the bucket -> **Empty** -> then **Delete** (you must type the bucket name to confirm).

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `BucketAlreadyExists` | The name is taken by someone else, anywhere in the world | Add a random suffix, e.g. `my-aws-bucket-ab12cd34` |
| `BucketAlreadyOwnedByYou` | You already created that bucket | Use the existing bucket or pick a new name |
| `InvalidBucketName` | Uppercase letters, spaces, underscores or `@` used | Use lowercase letters, digits, `-` and `.` only; 3–63 chars |
| `IllegalLocationConstraintException` | Region flag and location constraint disagree | Match `--region` and `LocationConstraint`; omit the constraint for `us-east-1` |
| `AccessDenied` when opening the Object URL | Bucket is private (expected) | Use a presigned URL, or configure a bucket policy only when public access is genuinely required |
| `BucketNotEmpty` on delete | Objects (or versions) still inside | Empty the bucket first; if versioning is on, delete all versions and delete markers |
| `AccessDenied` on create | IAM user lacks `s3:CreateBucket` | Attach an S3 policy to your IAM user/role |

## Key Takeaways
- S3 = Simple Storage Service, an object store made of **buckets** and **objects**.
- Bucket names are **globally unique**, lowercase, 3–63 characters.
- Buckets live in a region; "folders" are just key prefixes.
- Highly durable, scalable, pay-as-you-go with no upfront cost.
- ACLs are disabled by default — use IAM and bucket policies for access control.
- Anything you can do in the console you can do repeatably with Terraform (IaC), which is the next step.

## Next: [Terraform — S3 Bucket Create and Upload Files](./16-s3-bucket-create-and-upload.md)
