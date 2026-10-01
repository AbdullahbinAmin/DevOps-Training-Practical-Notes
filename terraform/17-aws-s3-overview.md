# Chapter 17 — AWS S3 Overview (and manual bucket)

## Objective
Understand what S3 is and create a bucket manually.

## What is S3?
- **S3 = Simple Storage Service.**
- You can think of it as a **bucket in the cloud** where you store data or files.
- Store and retrieve any amount of data from anywhere.
- Pricing: no minimum fee; you pay as you use (storage, requests, data transfer). ⚠️ Verification Required: check the pricing section on the S3 page.
- Common uses: backup and restore, disaster recovery, data archives, file storage, static websites.
- Vocabulary: S3 **bucket** (the container) and **object** (a file inside the bucket).

## Prerequisites
- IAM user login.

## Step 1 — Open S3
Search `S3` in the console search bar and open it. You will see the S3 page and the **Create bucket** button.

## Step 2 — Create a bucket (GUI)

**Step 1:** Click **Create bucket**.

**Step 2:** **Bucket type / General configuration:** choose **General purpose**.

**Step 3:** **Bucket name:** the name must be **globally unique** across all AWS accounts in the world.

Naming rules: only lowercase letters, numbers and hyphens (`-`), no spaces or special characters, 3–63 characters.

Try something unique, for example `abcd12345xyz-yourname`. If AWS says the name is taken, change it.

**Step 4:** **Object Ownership:** **ACLs disabled (recommended)**.

**Step 5:** Keep **Block all public access** as it is (ticked) for now.

**Step 6:** **Bucket Versioning:** **Disable** (versioning can add extra cost).

**Step 7:** Click **Create bucket**.

**Expected:** a green message "Successfully created bucket". Your bucket appears in the list with the region shown.

## Step 3 — Look inside
1. Click the bucket name. It is empty.
2. You can click **Create folder** to make a folder, or **Upload** to add files.

## Cleanup
Empty and delete the bucket if you do not need it: select the bucket → **Empty** (type `permanently delete`) → then **Delete** (type the bucket name).

## Troubleshooting

### Error 1
```text
Bucket with the same name already exists
```
**Reason:** Someone in the world already uses that name.
**Fix:** Change the name (add numbers or your name).

### Error 2
```text
Bucket name is not valid (uppercase / special characters)
```
**Fix:** Use only lowercase letters, numbers and hyphens.

## Final Result
You know what S3 is and created a bucket through the console. In the next chapter you do it with Terraform.
