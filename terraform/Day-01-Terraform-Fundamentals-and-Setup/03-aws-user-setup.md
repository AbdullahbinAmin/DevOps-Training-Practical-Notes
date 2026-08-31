# Day 01 — AWS User Setup (IAM)

> Stop using the root user: enable MFA on it, then create a dedicated IAM user (`tf-user`) in a group, secure it with MFA, and sign in as that user.

## Learning Objectives
- Explain what IAM manages: users, groups, roles, policies, and permissions.
- Enable MFA on the AWS root account using an authenticator app.
- Create an IAM user inside a group and attach permissions to the group, not the user.
- Sign in through the account-specific console URL as an IAM user.
- Apply least-privilege thinking instead of reaching for `AdministratorAccess` by reflex.

## Prerequisites
- An AWS account with root (email + password) access.
- A phone with an authenticator app: Google Authenticator, Microsoft Authenticator, Authy, or similar.
- A password manager or other safe place to store credentials.

## Concept

**IAM (Identity and Access Management)** is the AWS service that answers "who is allowed to do what". It manages Users, Groups, Roles, Policies, and Security settings such as MFA.

The **root user** is the email address you signed up with. It has unrestricted access to everything including billing and account closure. Using it for daily work is **not good practice** — one leaked root password compromises the whole account, and root permissions cannot be reduced.

So the pattern is:

```
Root user  ->  used once, to set up MFA and create IAM users, then locked away
    |
    v
IAM Group "admins"  <- policies attached HERE
    |
    v
IAM User "tf-user"  <- you sign in as this every day, with MFA
```

Attaching policies to a **group** rather than each user means new teammates just get added to the group and instantly have the right access.

| Common AWS managed policy | What it grants |
|---|---|
| `AdministratorAccess` | Full access to all AWS services (use only for training accounts) |
| `PowerUserAccess` | Full access except IAM, billing, and account management |
| `ReadOnlyAccess` | Read-only access to AWS services |

For a personal training account `AdministratorAccess` is acceptable. In a real workplace, grant only the services the user actually needs.

## Step-by-Step Practical

### Part A — Open IAM and secure the root user

1. Sign in to the AWS Management Console as the **root user**, then search for **IAM** in the console search bar and open **IAM (Identity and Access Management)**.

2. On the IAM Dashboard note the left navigation: **Users, Groups (User groups), Roles, Policies**, plus the security recommendations panel.

3. Enable MFA for the root user:
   - On the IAM Dashboard, find the root user security recommendation and click **Add MFA**.
   - **Device name:** `My Mobile`
   - **Choose method:** **Authenticator app** (recommended). The other options are Hardware TOTP token and Security key.
   - Click **Next**, scan the QR code with your authenticator app.
   - Enter **two consecutive codes** from the app to enable MFA.

### Part B — Create the IAM user and group

4. Go to **IAM → Users → Create user**.
   - **User name:** `tf-user`
   - Tick **Provide user access to the AWS Management Console**.
   - **Password type:** **Custom password** (set a strong one).
   - Leave **Require password change on next login** unticked for a training account.
   - Click **Next**.

5. On **Set permissions**, choose **Add user to group** → **Create group**.
   - **New group name:** `admins`
   - Under **Attach policies**, tick **AdministratorAccess**.
   - Click **Create group**, then **Next**.

6. **Review** the summary and click **Create user**.

7. On the success screen you receive four things. Save them securely — the console password is shown only once.

```
User name        : tf-user
Console password : (click "Show" to reveal)
Console sign-in URL : https://<YOUR_ACCOUNT_ID>.signin.aws.amazon.com/console
Account ID       : YOUR_ACCOUNT_ID   (12 digits)
```

Never paste real values into notes, chat, or git. Use placeholders like `YOUR_ACCOUNT_ID` and `YOUR_ACCESS_KEY` when sharing.

### Part C — Sign in as the IAM user

8. Sign out of the root user, then open the **Console sign-in URL** from step 7.

9. Sign in with:
   - **Account ID:** `YOUR_ACCOUNT_ID` (pre-filled if you used the sign-in URL)
   - **User name:** `tf-user`
   - **Password:** the custom password you set
   - Optionally tick **Remember this account**.

### Part D — Enable MFA for the IAM user

10. Go to **IAM → Users → tf-user → Security credentials → Multi-factor authentication (MFA) → Assign MFA device**.

11. Repeat the same flow: **Authenticator app** → scan the QR code → enter two consecutive codes → **Add MFA**. `tf-user` is now MFA-protected.

12. Optional — once the AWS CLI is set up (next notes), you can confirm the same facts from the terminal.

```bash
aws sts get-caller-identity
```

```bash
aws iam list-users
```

## Expected Output

`aws sts get-caller-identity` for `tf-user` returns:

```json
{
    "UserId": "AIDAEXAMPLE1111",
    "Account": "YOUR_ACCOUNT_ID",
    "Arn": "arn:aws:iam::YOUR_ACCOUNT_ID:user/tf-user"
}
```

In the console, the IAM Dashboard security panel should show root MFA enabled, and `tf-user` should list one assigned MFA device.

## Verification

1. The top-right of the console shows `tf-user @ YOUR_ACCOUNT_ID`, **not** the root email address.
2. **IAM → Users → tf-user → Groups** lists `admins`.
3. **IAM → User groups → admins → Permissions** lists `AdministratorAccess`.
4. Signing out and back in prompts for an MFA code.
5. **IAM → Dashboard** no longer warns that root MFA is missing.
6. **IAM → Users → tf-user** shows **no access keys yet** — those come in note 05.

## Cleanup

Keep `tf-user` for the rest of the course. Only remove it when the training is over, and remove dependencies first:

1. Delete any access keys under **tf-user → Security credentials**.
2. Deactivate and remove the MFA device.
3. Remove the user from the `admins` group.
4. Delete the user, then the group.

```bash
aws iam remove-user-from-group --user-name tf-user --group-name admins
aws iam delete-login-profile --user-name tf-user
aws iam delete-user --user-name tf-user
```

Leave root MFA enabled permanently — that is not a lab artifact, it is basic account hygiene.

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Your authentication information is incorrect` at sign-in | Trying to sign in as an IAM user on the root sign-in page | Use the account-specific console sign-in URL, or click "Sign in using IAM user credentials" |
| MFA setup fails: "codes do not match" | Phone clock drift, or codes entered too fast | Enable automatic time on the phone; wait for a fresh code and enter two **consecutive** codes |
| `AccessDenied` after signing in as `tf-user` | User is not actually in the group, or the policy was never attached | Check tf-user → Groups, and admins → Permissions |
| Password shown once and lost | Console password is only displayed at creation | tf-user → Security credentials → **Manage** console password to reset it |
| Locked out after losing the MFA phone | No backup device or recovery path | Sign in as root to reset the IAM user's MFA; for root MFA loss use AWS account recovery |
| `You must specify a value for MFA` on CLI calls | Policy requires MFA for API calls | Use `aws sts get-session-token --serial-number <mfa-arn> --token-code 123456` |

## Key Takeaways
- Use the root user only to bootstrap: enable its MFA, create IAM users, then stop using it.
- Attach policies to groups; put users in groups. It scales, per-user policies do not.
- MFA on both root and IAM users is the single highest-value security step you can take today.
- `AdministratorAccess` is fine for a personal training account and wrong for production — grant least privilege.
- Save the console sign-in URL and Account ID; you will need them every day.

## Next: [AWS CLI Setup](04-aws-cli-setup.md)
