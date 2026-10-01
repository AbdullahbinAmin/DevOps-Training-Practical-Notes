# Chapter 6 — AWS User Setup (IAM User + MFA)

## Objective
Stop using the root user. Add MFA to root, then create an IAM user in an admin group and use that user for all work.

## Prerequisites
- AWS account from Chapter 5
- A phone with an authenticator app (Google Authenticator, Microsoft Authenticator, etc.)

## Architecture / Flow
```text
Root user → (add MFA) → create Group "admins" → create IAM user "tf-user" (in the group)
        → sign out → sign in as tf-user → add MFA for tf-user
```

## Part A — Add MFA to the root user (GUI)

**Step 1:** Sign in to the AWS Console as **root user**.

**Step 2:** In the top search bar type `IAM` and open **IAM**.

**Step 3:** On the IAM dashboard you will see a warning about MFA for the root user. Click **Add MFA**.

**Step 4:** Enter:
- **Device name:** for example `mobile`
- **MFA device type:** **Authenticator app**

Click **Next**.

**Step 5:** On your phone open the authenticator app → **Add account → Scan QR code**. Scan the QR code shown on the screen.

**Step 6:** Type two consecutive 6-digit codes from the app into **MFA code 1** and **MFA code 2**. Click **Add MFA**.

**Expected:** the console says the MFA device was assigned.

## Part B — Create a group with admin permission

**Step 1:** IAM → **User groups** → **Create group**.

**Step 2:** **User group name:** `admins`

**Step 3:** In **Attach permissions policies**, search `AdministratorAccess` and tick it. (Its description: provides full access to AWS services.)

**Step 4:** Click **Create user group**.

## Part C — Create the IAM user (GUI)

**Step 1:** IAM → **Users** → **Create user**.

**Step 2:** **User name:** `tf-user` (you can use any name).

**Step 3:** Tick **Provide user access to the AWS Management Console**.

**Step 4:** Select **I want to create an IAM user**.

**Step 5:** Choose **Custom password** and type a strong password. Untick **Users must create a new password at next sign-in** (as in the video). Click **Next**.

**Step 6:** In **Set permissions** choose **Add user to group**, tick `admins`, click **Next**.

**Step 7:** (Optional) Add a tag. Click **Create user**.

**Step 8 — Save these details in a safe place (very important):**
- **Console sign-in URL**
- **User name**
- **Password**
- **AWS account ID** (the 12-digit number in the sign-in URL)

Note: UI text and order of screens can change between AWS console versions. ⚠️ Verification Required if your screen looks different.

## Part D — Sign in as the new user

**Step 1:** Click your account name (top right) → **Sign out**.

**Step 2:** Open the **Console sign-in URL** you saved.

**Step 3:** Choose **IAM user**. Enter **Account ID**, **user name** and **password**. Tick **Remember this account** (optional). Click **Sign in**.

**Expected:** top right shows your IAM user name (for example `tf-user @ 1234-5678-9012`), not root.

## Part E — Add MFA to the IAM user

**Step 1:** Search `IAM` → open **IAM**. Your dashboard offers **Add MFA for yourself**. Click it.

**Step 2:** **Assign MFA** → device name `admin` (any name) → **Authenticator app** → **Next**.

**Step 3:** Scan the QR code with the app. Enter two consecutive codes. Click **Add MFA**.

**Expected:** MFA is enabled for the IAM user.

## Troubleshooting

### Error 1
```text
Invalid MFA code
```
**Reason:** Phone time is wrong, or you reused the same code twice.
**Fix:** Turn on automatic time on the phone. Wait for a new code for the second field.

### Error 2
Cannot sign in as IAM user.
**Reason:** Wrong account ID or you selected **Root user** instead of **IAM user**.
**Fix:** Use the saved sign-in URL and choose **IAM user**.

## Final Result
- Root user has MFA.
- IAM user `tf-user` (in group `admins`) can sign in with MFA.
- You have saved the sign-in URL, username, password and account ID.

Security note: `AdministratorAccess` is fine for learning. In real jobs give only the permissions required.
