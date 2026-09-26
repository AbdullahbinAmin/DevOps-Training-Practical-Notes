# Chapter 14 — Domain Name Mapping (DNS A Records)

## Objective
Map a real, purchased domain name to your EC2 instance's public IP address so that your application (deployed in earlier chapters) can be accessed via a friendly domain name instead of a raw IP address.

## Prerequisites
* OS: Ubuntu EC2 instance running your application (e.g., the Expense Tracker from Chapter 13)
* Required software: None additional (this is done via a domain registrar's web console)
* Required account: An account with a domain registrar (example used: GoDaddy)
* Required: A running application accessible via `http://<EC2_PUBLIC_IP>:<PORT>`

## Concept — What is DNS Mapping?

**The problem:**
Right now, your application is only reachable using a hard-to-remember IP address, such as `http://34.201.xx.xx:8080`. This is not user-friendly and will change if your EC2 instance is stopped/restarted without an Elastic IP.

**The solution:**
You purchase a **domain name** (e.g., `trainwithshubham.com`) and create a **DNS record** that points a specific subdomain (like `docker.trainwithshubham.com`) to your server's IP address. Once configured, visiting that domain in a browser routes the request to your server's IP automatically.

## Step 1 — Purchase a Domain Name
1. Go to a domain registrar (example used: GoDaddy — https://www.godaddy.com).
2. Create an account/profile if you don't have one.
3. Search for an available domain name (a `.online`/`.xyz`/similar cheap domain works fine for testing/learning purposes — often available for a very small price, like $0.01–$1 for the first year).
4. Purchase the domain.

> ⚠️ Note: Registrars and pricing change over time — always check current pricing/availability directly on the registrar's site.

## Step 2 — Access DNS Management
1. Log in to your registrar account.
2. Go to your **Domains** list.
3. Click **Manage DNS** (or "DNS Management" / "DNS Settings," depending on the registrar) for your purchased domain.

## Step 3 — Add an "A" Record

**What an A Record is:**
An "A" (Address) record is the most basic type of DNS record. It maps a hostname/subdomain directly to an IPv4 address.

1. Click **Add a New Record** (or similar button).
2. **Type:** Select **A**.
3. **Name/Host:** Enter your desired subdomain, for example: `docker` (this will create `docker.yourdomain.com`).
4. **Value/Points to:** Enter your **EC2 instance's Public IPv4 address**.
5. **TTL (Time to Live):** You can leave the default, or set a custom value like **600 seconds** (this controls how long DNS resolvers cache this record before re-checking it — a shorter TTL means changes propagate faster, useful while testing).
6. Click **Save**.

**Expected result:** The registrar shows a message like "Updating DNS records" — this can take anywhere from a few seconds to a few minutes (sometimes longer, depending on DNS propagation) to fully take effect worldwide.

## Step 4 — Test the Domain
Open your browser and go to:
```text
http://docker.yourdomain.com:8080
```
(Replace `docker.yourdomain.com` with your actual subdomain, and `8080` with whichever port your application uses.)

**Expected result:** Your application loads — the same way it did when using the raw IP address.

> ⚠️ If it does not load immediately, wait a few minutes for DNS propagation and try again. Using an **incognito/private browser window** helps avoid stale DNS caching issues on your own computer while testing.

## Step 5 (Optional but Recommended) — Update Your Nginx Configuration to Use the Domain Name

If your project (like Chapter 12's project) uses Nginx as a reverse proxy, Nginx's configuration may specify `server_name localhost;` by default. If you want Nginx to correctly recognize your new domain as the expected server name:

```bash
nano nginx/default.conf
```
Update the `server_name` line:
```nginx
server {
    listen 80;
    server_name docker.yourdomain.com;

    location / {
        proxy_pass http://notes_container:8000;
    }
}
```

**Why this matters:** Without this, Nginx may still work by default (since it often falls back to matching any server name when nothing else matches), but explicitly setting `server_name` is the correct, production-ready practice and avoids ambiguity if you later host multiple domains/apps behind the same Nginx server.

**Rebuild and restart:**
```bash
docker compose down
docker compose up -d --build
```

**Verify:**
```bash
docker ps
```
Then access your domain again in the browser.

## Step 6 (Optional) — Add a Second A Record Pointing to the Same IP
You can add multiple subdomains pointing to the same server if needed (for example, `docker` and `demo` both pointing to the same IP), each with their own A record entry, following the same steps as Step 3.

## Concept — HTTPS Note
By default, a plain domain mapped only with an A record will load over `http://`, not `https://`. Browsers may show a "Not Secure" or "connection is not private" warning when you first try `https://`. To properly enable HTTPS, you would need to set up an SSL/TLS certificate (for example, using **Let's Encrypt** with a tool like **Certbot**, or terminating SSL at a load balancer) — this is outside the scope of this chapter but is a natural next step for a production deployment.

If prompted with a browser warning about an insecure connection while testing over plain HTTP, you can proceed by clicking "Continue to site" / "Proceed anyway" for learning/testing purposes only — never do this for a real production site without proper SSL.

## Troubleshooting

### Error 1
Domain does not load at all, even after waiting.
**Reason:** DNS record not fully propagated yet, or a typo in the IP address entered.

**Fix:** Double-check the IP address value in the A record matches your EC2 instance's current Public IPv4 exactly, and wait longer (DNS propagation can occasionally take up to 24–48 hours in rare cases, though usually much faster).

### Error 2
```text
This site can't provide a secure connection / doesn't support a secure connection
```
**Reason:** You tried accessing via `https://` but only HTTP is configured (no SSL certificate set up).

**Fix:** Use `http://` instead of `https://` for now, or set up SSL via Certbot/Let's Encrypt for a proper HTTPS setup.

### Error 3
Domain works with `http://IP:PORT` but not with the domain name.
**Reason:** The Security Group might restrict access by some other rule, or the A record still points to an old/incorrect IP (e.g., if the EC2 instance was restarted and got a new IP without an Elastic IP attached).

**Fix:** Re-verify your EC2 instance's CURRENT public IP and update the A record if it has changed. Consider attaching an **Elastic IP** to your instance so the IP address never changes across restarts.

## Cleanup
No cleanup required — domain mapping is a one-time configuration that can remain in place.

## Final Result
* A purchased domain (or subdomain) now points to your EC2 instance's public IP via a DNS A record.
* Your application is accessible via a friendly URL like `http://docker.yourdomain.com:8080` instead of a raw IP address.
* (Optional) Your Nginx configuration is updated to explicitly recognize the domain name.

## What's Next
Chapter 15 (Bonus) covers two additional Docker tools: **Docker Scout** (image vulnerability scanning) and **Docker Init** (auto-generating Dockerfile/Compose boilerplate).
