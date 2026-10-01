# Chapter 21 — Project: Host a Static Website Using Terraform

## Objective
Publish a static website (HTML + CSS) on the internet using S3 and Terraform. First understand the steps manually, then automate them.

## Prerequisites
- Chapters 9, 18, 19 completed.
- `.env` credentials in `tf-aws`.
- Your region code.

## Project flow (the 6 steps)
```text
1. Select provider (aws + random)
2. Create S3 bucket
3. Allow public access (public access block settings)
4. Add bucket policy (allow public read)
5. Configure website hosting
6. Upload files (index.html, styles.css)
7. Output the website URL
```

## Step 1 — Project folder and website files
```bash
source .env
mkdir project-static-website
cd project-static-website
```

Create `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My Terraform Website</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <h1>Hello from Terraform and S3!</h1>
  <p>This website is hosted on an AWS S3 bucket.</p>
</body>
</html>
```

Create `styles.css`:

```css
body {
  font-family: Arial, sans-serif;
  text-align: center;
  background: linear-gradient(135deg, #4facfe, #00f2fe);
  color: #ffffff;
  padding-top: 80px;
}
```
(You can also use any simple sample website. The video used a sample site from a public GitHub repo with the same two file names.)

Verify:
```bash
ls
```
**Expected:** `index.html` and `styles.css`.

## Part A — Do it manually first (to understand)

1. S3 → **Create bucket** → General purpose → unique name (for example `myweb-abcd12345-yourname`).
2. In **Block Public Access settings for this bucket**, **untick "Block all public access"** and tick the box **"I acknowledge that the current settings might result in this bucket and the objects within becoming public."**
3. Create the bucket.
4. Open the bucket → **Upload** → **Add files** → select `index.html` and `styles.css` → **Upload**.
5. Click `index.html` → find the **Object URL** → click it. You get **Access Denied**. Reason: public read permission is missing.
6. Fix: open the AWS documentation page **"Setting permissions for website access"** (search in AWS S3 docs). Copy the sample policy.
7. Bucket → **Permissions** tab → **Bucket policy** → **Edit** → paste this (replace the bucket name):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::<YOUR_BUCKET_NAME>/*"]
    }
  ]
}
```
- `<YOUR_BUCKET_NAME>` → your bucket name. Always re-check copied policies.

8. **Save changes**. Refresh the Object URL — the website appears.

**Lesson:** two things were needed: (1) allow public access (unblock), (2) add a bucket policy. For a real *website endpoint* we also need website hosting enabled, and Terraform will do that.

**Delete the manual bucket** (Empty → Delete) before the Terraform part to avoid confusion.

## Part B — Now with Terraform

### Step 1 — Create `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "<YOUR_REGION>"
}

# 1) Random ID for a unique bucket name
resource "random_id" "rand_id" {
  byte_length = 8
}

# 2) Bucket
resource "aws_s3_bucket" "my_webapp_bucket" {
  bucket = "my-webapp-bucket-${random_id.rand_id.hex}"
}

# 3) Public access settings (all false = allow public policy)
resource "aws_s3_bucket_public_access_block" "my_webapp_bucket" {
  bucket = aws_s3_bucket.my_webapp_bucket.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# 4) Bucket policy (public read)
resource "aws_s3_bucket_policy" "webapp_policy" {
  bucket = aws_s3_bucket.my_webapp_bucket.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "arn:aws:s3:::${aws_s3_bucket.my_webapp_bucket.id}/*"
      }
    ]
  })

  depends_on = [aws_s3_bucket_public_access_block.my_webapp_bucket]
}

# 5) Static website configuration
resource "aws_s3_bucket_website_configuration" "my_webapp" {
  bucket = aws_s3_bucket.my_webapp_bucket.id

  index_document {
    suffix = "index.html"
  }
}

# 6) Upload files (content_type is required for the browser to display them)
resource "aws_s3_object" "index_html" {
  bucket       = aws_s3_bucket.my_webapp_bucket.id
  source       = "index.html"
  key          = "index.html"
  content_type = "text/html"
}

resource "aws_s3_object" "styles_css" {
  bucket       = aws_s3_bucket.my_webapp_bucket.id
  source       = "styles.css"
  key          = "styles.css"
  content_type = "text/css"
}

# 7) Print the website URL
output "website_url" {
  value = aws_s3_bucket_website_configuration.my_webapp.website_endpoint
}
```

Replace `<YOUR_REGION>` with your region, for example `eu-north-1`.

**Where these resources come from:** Open the AWS provider docs (registry.terraform.io → hashicorp/aws → Documentation) and search `s3`. On the left you find `aws_s3_bucket`, `aws_s3_bucket_public_access_block`, `aws_s3_bucket_policy`, `aws_s3_bucket_website_configuration`, `aws_s3_object`. This is the way to search for any resource yourself.

**Explanation of the important parts:**
- `public_access_block` with all `false` = the same as unticking "Block all public access" in the console.
- `jsonencode({...})` converts HCL to JSON. In the video, the JSON policy was copied from AWS docs, then quotes were removed, `:` was replaced with `=`, and the bucket name was replaced by a reference. That is exactly what is written above.
- `depends_on` makes sure the public access settings are applied **before** the policy (otherwise AWS can reject the policy).
- `content_type` tells the browser the file type (see troubleshooting).
- Resource names cannot contain dots, so we use `index_html` instead of `index.html`.

### Step 2 — Run
```bash
terraform init
terraform validate
terraform apply
```
Type `yes`.

**Expected end:**
```text
Apply complete! Resources: 7 added ...

Outputs:
website_url = "my-webapp-bucket-....s3-website.<region>.amazonaws.com"
```
(Depending on region the separator may be `s3-website.` or `s3-website-`. Use exactly what Terraform prints.)

### Step 3 — Verify in the console
- S3 → your bucket → **Objects**: `index.html`, `styles.css`.
- **Permissions**: **Block public access = Off**, and the **Bucket policy** is present.
- **Properties** → scroll to the end → **Static website hosting: Enabled** with a website URL.

### Step 4 — Test in the browser
Copy the `website_url` output. Open a new tab and paste it. Add `http://` in front if your browser needs it:

```text
http://<website_url_output>
```
**Expected:** your website page opens.

**Note:** S3 website endpoints use **HTTP only** (not HTTPS).

## Troubleshooting

### Error 1 — The file downloads instead of opening
**Reason:** No `content_type`, so the browser does not know it is HTML.
**Fix:** Add `content_type = "text/html"` (and `text/css` for CSS), then `terraform apply` again.

### Error 2 — `Access Denied` (403) on the website URL
**Reason:** Policy or public access settings missing or not applied.
**Fix:** Confirm `aws_s3_bucket_public_access_block` has all four values `false`, and the policy exists. Run `terraform apply` again.

### Error 3
```text
Error: Invalid resource name
```
**Reason:** A resource label contains a dot (for example `index.html`).
**Fix:** Use underscores (`index_html`).

### Error 4
```text
AccessDenied: ... BlockPublicPolicy
```
**Reason:** The policy was applied before public access was unblocked.
**Fix:** Keep the `depends_on` line and run `terraform apply` again. Also check that your AWS account level "Block Public Access" (S3 → Block Public Access settings for this account) is not turned on. ⚠️ Verification Required.

## Cleanup
```bash
terraform destroy
```
Type `yes`. (Terraform deletes the objects it uploaded; if the bucket refuses because of extra files, empty it in the console and run destroy again.)

## Final Result
The website opens in the browser using the URL printed by Terraform.

## Summary of resources used
`random_id` → `aws_s3_bucket` → `aws_s3_bucket_public_access_block` → `aws_s3_bucket_policy` → `aws_s3_bucket_website_configuration` → `aws_s3_object` (x2) → `output`
