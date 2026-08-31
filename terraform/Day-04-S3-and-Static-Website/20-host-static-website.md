# Day 04 — Project: Host a Static Website Using Terraform

> Deploy a public static website on S3 with Terraform: bucket, public access block, bucket policy, website configuration, uploaded HTML/CSS, and an output URL.

## Learning Objectives
- Reproduce the manual "S3 static website" steps as Terraform code.
- Use `aws_s3_bucket_website_configuration` for index and error documents.
- Understand how `aws_s3_bucket_public_access_block` and `aws_s3_bucket_policy` work together to allow public reads.
- Explain the security implications of a public bucket.
- Output and open the live website endpoint.

## Prerequisites
- Completed lessons 15, 16 and 17 (S3 basics, bucket + upload, random provider).
- Terraform 1.x, AWS provider 5.x, credentials configured.
- IAM permissions for `s3:CreateBucket`, `s3:PutObject`, `s3:PutBucketPolicy`, `s3:PutBucketPublicAccessBlock`, `s3:PutBucketWebsite`, plus the matching deletes.

## Concept (plain English, beginner friendly)
A **static website** is plain files — HTML, CSS, JS, images — with no server-side code. S3 can serve those files directly over HTTP, which makes it one of the cheapest ways to host a site.

Doing it by hand takes five steps: create bucket -> upload files -> allow public access -> enable static website hosting -> get the URL. Terraform turns those into five resources you can apply, destroy and re-apply at will.

Four things must all be true before the site is reachable:

1. **The files exist** in the bucket (`aws_s3_object`).
2. **Public access block is relaxed** (`aws_s3_bucket_public_access_block`). AWS blocks public access by default. Two settings must be `false`: `block_public_policy` (otherwise your policy is rejected) and `restrict_public_buckets` (otherwise anonymous requests are refused). Leave the two **ACL** settings `true` — you do not need ACLs, and keeping them blocked is safer.
3. **A bucket policy grants anonymous read** (`aws_s3_bucket_policy`) with `Principal: "*"` and `s3:GetObject` on `arn:aws:s3:::<bucket>/*`. This is the modern replacement for `acl = "public-read"`, which no longer works because ACLs are disabled by default.
4. **Website hosting is configured** (`aws_s3_bucket_website_configuration`) naming an index document and an error document.

Also important: each object needs the right **`content_type`**. If `index.html` is uploaded as `binary/octet-stream`, browsers download it instead of rendering it.

### Security implication — read this before you apply
Making a bucket public means **anyone on the internet can read every object in it**, with no authentication, forever, and they can crawl it. Consequences:

- **Never put anything sensitive in a public bucket** — no config with secrets, no customer data, no backups, no `.env`, no `.git` folder.
- Use a **dedicated bucket** for the website. Do not add a public policy to a bucket that also holds private data.
- **You pay for their requests.** Public egress and GET requests are billable; a traffic spike or scraping is your bill.
- Grant **only `s3:GetObject`** — never `s3:PutObject`, `s3:DeleteObject` or `s3:*` to `Principal: "*"`. A public-write bucket gets used to host malware.
- The S3 website endpoint is **HTTP only**. For HTTPS, a custom domain, and to keep the bucket private, front it with **CloudFront + Origin Access Control** — that is the production pattern.
- AWS will show a red **"Publicly accessible"** warning on this bucket. That warning is correct and expected here; treat it as a prompt to confirm the bucket contains only website files.

## Step-by-Step Practical

1. Create the project folder.

```bash
mkdir -p static-website
cd static-website
```

Target structure:

```text
static-website/
├── main.tf
├── outputs.tf
└── website/
    ├── index.html
    ├── error.html
    └── styles.css
```

2. Create the website files. `website/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My Static Website</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <main class="card">
    <h1>Hello World!</h1>
    <p>Welcome to my Static Website</p>
    <p class="muted">Hosted on Amazon S3 and deployed with Terraform.</p>
  </main>
</body>
</html>
```

3. `website/error.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>404 — Page Not Found</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <main class="card">
    <h1>404</h1>
    <p>Sorry, that page does not exist.</p>
    <p><a href="/index.html">Back to home</a></p>
  </main>
</body>
</html>
```

4. `website/styles.css`:

```css
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: system-ui, -apple-system, "Segoe UI", Arial, sans-serif;
  background: linear-gradient(135deg, #1e3a8a, #0ea5e9);
  color: #0f172a;
  padding: 1.5rem;
}

.card {
  background: #ffffff;
  padding: 3rem 2.5rem;
  border-radius: 14px;
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.25);
  text-align: center;
  max-width: 34rem;
}

h1 { font-size: 2.5rem; color: #1e3a8a; margin-bottom: 0.75rem; }

p { font-size: 1.05rem; line-height: 1.6; margin-bottom: 0.5rem; }

.muted { color: #64748b; font-size: 0.9rem; }

a { color: #0ea5e9; }
```

5. Create `main.tf`.

```hcl
# main.tf

terraform {
  required_version = ">= 1.5.0"

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
  region = var.aws_region
}

variable "aws_region" {
  description = "AWS region to deploy the website bucket into"
  type        = string
  default     = "ap-northeast-1"
}

variable "bucket_prefix" {
  description = "Prefix for the globally unique bucket name"
  type        = string
  default     = "my-web-app"
}

# Random suffix so the bucket name is globally unique
resource "random_id" "suffix" {
  byte_length = 4
}

# ---------------------------------------------------------------
# 1. The bucket that holds the website files
# ---------------------------------------------------------------
resource "aws_s3_bucket" "my_web_app" {
  bucket = "${var.bucket_prefix}-${random_id.suffix.hex}"

  tags = {
    Name      = "static-website"
    Purpose   = "public-static-site"
    ManagedBy = "terraform"
  }
}

# ---------------------------------------------------------------
# 2. Relax Block Public Access so a public read policy is allowed
#    ACL settings stay TRUE - we do not use ACLs at all.
# ---------------------------------------------------------------
resource "aws_s3_bucket_public_access_block" "my_web_app" {
  bucket = aws_s3_bucket.my_web_app.id

  block_public_acls       = true  # keep ACLs blocked
  ignore_public_acls      = true  # keep ACLs blocked
  block_public_policy     = false # required: allow a public bucket policy
  restrict_public_buckets = false # required: allow anonymous GETs
}

# ---------------------------------------------------------------
# 3. Bucket policy: anonymous READ-ONLY access to objects
# ---------------------------------------------------------------
resource "aws_s3_bucket_policy" "my_web_app" {
  bucket = aws_s3_bucket.my_web_app.id
  policy = data.aws_iam_policy_document.public_read.json

  # The policy is rejected unless the access block is relaxed first
  depends_on = [aws_s3_bucket_public_access_block.my_web_app]
}

data "aws_iam_policy_document" "public_read" {
  statement {
    sid    = "PublicReadGetObject"
    effect = "Allow"

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions   = ["s3:GetObject"] # READ ONLY - never add Put/Delete here
    resources = ["${aws_s3_bucket.my_web_app.arn}/*"]
  }
}

# ---------------------------------------------------------------
# 4. Enable static website hosting
# ---------------------------------------------------------------
resource "aws_s3_bucket_website_configuration" "my_web_app" {
  bucket = aws_s3_bucket.my_web_app.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

# ---------------------------------------------------------------
# 5. Upload the website files with correct content types
# ---------------------------------------------------------------
locals {
  content_types = {
    html = "text/html"
    css  = "text/css"
    js   = "application/javascript"
    png  = "image/png"
    jpg  = "image/jpeg"
    svg  = "image/svg+xml"
    ico  = "image/x-icon"
  }

  website_files = fileset("${path.module}/website", "**")
}

resource "aws_s3_object" "website_files" {
  for_each = local.website_files

  bucket = aws_s3_bucket.my_web_app.id
  key    = each.value
  source = "${path.module}/website/${each.value}"
  etag   = filemd5("${path.module}/website/${each.value}")

  content_type = lookup(
    local.content_types,
    lower(reverse(split(".", each.value))[0]),
    "application/octet-stream"
  )
}
```

The equivalent policy as raw JSON, if you prefer seeing it that way (this is what `data.aws_iam_policy_document` renders):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-web-app-a1b2c3d4/*"
    }
  ]
}
```

6. Create `outputs.tf` to surface the website URL.

```hcl
# outputs.tf

output "bucket_name" {
  description = "Name of the website bucket"
  value       = aws_s3_bucket.my_web_app.bucket
}

output "website_endpoint" {
  description = "S3 static website endpoint (HTTP only)"
  value       = aws_s3_bucket_website_configuration.my_web_app.website_endpoint
}

output "website_url" {
  description = "Full URL to open in a browser"
  value       = "http://${aws_s3_bucket_website_configuration.my_web_app.website_endpoint}"
}

output "uploaded_files" {
  description = "Files uploaded to the bucket"
  value       = sort([for o in aws_s3_object.website_files : o.key])
}
```

7. Deploy.

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply    # type: yes
```

8. Open the site.

```bash
terraform output -raw website_url

# Linux / macOS
xdg-open "$(terraform output -raw website_url)" 2>/dev/null || open "$(terraform output -raw website_url)"

# Windows (Git Bash)
start "$(terraform output -raw website_url)"
```

## Expected Output

```text
Plan: 8 to add, 0 to change, 0 to destroy.

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

bucket_name      = "my-web-app-a1b2c3d4"
uploaded_files   = [
  "error.html",
  "index.html",
  "styles.css",
]
website_endpoint = "my-web-app-a1b2c3d4.s3-website-ap-northeast-1.amazonaws.com"
website_url      = "http://my-web-app-a1b2c3d4.s3-website-ap-northeast-1.amazonaws.com"
```

In the browser: a white card on a blue gradient reading **Hello World!** / *Welcome to my Static Website*.

Note the endpoint format differs by region — older regions use `s3-website-<region>` (hyphen), newer ones `s3-website.<region>` (dot). Always take the value from the output rather than typing it.

## Verification

1. Open the website URL — the page renders (does not download).
2. Request a path that does not exist — you should get your `error.html`, not the S3 XML error:

```bash
URL="$(terraform output -raw website_url)"

curl -I "$URL"                     # expect HTTP/1.1 200 OK, Content-Type: text/html
curl -I "$URL/styles.css"          # expect Content-Type: text/css
curl -s -o /dev/null -w "%{http_code}\n" "$URL/no-such-page"   # expect 404 with error.html body
curl -s "$URL" | head -n 5         # expect the index.html markup
```

3. Console check: S3 -> bucket -> **Properties** -> **Static website hosting** shows *Enabled* with index `index.html`, error `error.html`. Under **Permissions** the bucket shows *Publicly accessible* and your `PublicReadGetObject` policy.
4. Idempotency: `terraform plan` reports **No changes**.
5. Edit `website/index.html`, then `terraform apply` — only that object updates (the `etag` detected the change).

## Cleanup

```bash
terraform destroy    # type: yes
```

```text
Destroy complete! Resources: 8 destroyed.
```

Because every object was created through Terraform, destroy empties the bucket for you. If you uploaded anything by hand:

```bash
aws s3 rm "s3://$(terraform output -raw bucket_name)/" --recursive
terraform destroy
```

Then confirm the URL no longer resolves and the bucket is gone from the console. Leaving a public bucket running is both a cost and a security liability, so clean up when the lab is finished.

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `AccessDenied` / `403 Forbidden` on the website URL | Bucket policy missing, or `restrict_public_buckets`/`block_public_policy` still `true` | Set both to `false` and apply the `aws_s3_bucket_policy` |
| `Error putting S3 policy: AccessDenied` | Policy applied before the access block was relaxed | Add `depends_on = [aws_s3_bucket_public_access_block...]` (already in the config) |
| `404 Not Found` on the root URL | Index document name mismatch (`index.htm` vs `index.html`) | Make `index_document.suffix` match the real filename exactly |
| Browser downloads the page instead of showing it | `content_type` is `binary/octet-stream` | Set `content_type` per file extension, as in the `locals` map |
| CSS not applied | `styles.css` uploaded with the wrong content type, or the `href` path is wrong | Ensure `text/css` and that `href="styles.css"` matches the object key |
| `AccessControlListNotSupported` | Tried `acl = "public-read"` | Remove `acl`; use the bucket policy — ACLs are disabled by default |
| `NoSuchWebsiteConfiguration` | Using the REST endpoint (`s3.amazonaws.com`) instead of the website endpoint | Use the `website_endpoint` output |
| `BucketAlreadyExists` | Bucket name collision | The `random_id` suffix prevents this; re-run after `terraform taint random_id.suffix` if needed |
| Site loads over HTTP but not HTTPS | S3 website endpoints do not support HTTPS | Put CloudFront in front with an ACM certificate |
| `BucketNotEmpty` on destroy | Objects added outside Terraform | `aws s3 rm s3://<bucket>/ --recursive`, then destroy |
| Stale content after an update | Browser cache, or missing `etag` so no re-upload | Hard refresh; keep `etag = filemd5(...)` |

## Key Takeaways
- Five resources cover the whole project: bucket, public access block, bucket policy, website configuration, and the objects.
- Public access needs **both** a relaxed public access block and a bucket policy — either alone gives 403.
- Grant only `s3:GetObject` to `Principal: "*"`, and keep the bucket dedicated to website files. Anything in a public bucket is world-readable.
- ACLs are disabled by default; policies are the modern mechanism.
- Correct `content_type` is what makes the browser render rather than download.
- `fileset` + `for_each` uploads a whole directory and keeps it in sync via `etag`.
- The S3 website endpoint is HTTP-only; use CloudFront + OAC for HTTPS, a custom domain, and a private origin.
- Terraform makes the whole site reproducible: one `apply` to deploy, one `destroy` to remove.

## Next: [Day 05 — Terraform Variables, Outputs and Remote State](../Day-05-Variables-and-Remote-State/README.md)
