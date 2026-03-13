# Email Configuration

This document covers the email setup for Heero Publishers, including the error we encountered and how to configure a real email service in the future.

---

## The Problem We Fixed

### Symptom
After a user submitted an article or registered an account, they saw a **"Server Error — A server error has occurred"** page. Despite the error, the submission/registration was actually saved in the database.

### Root Cause
The Django settings had `EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"` configured, but **no SMTP server credentials were set up**. Janeway tries to send a notification email (e.g., "submission received" or "registration confirmation") at the end of these actions. When the SMTP connection fails, Django throws an unhandled exception, resulting in a 500 Server Error.

### The Fix
We changed the email backend to **console mode**, which logs emails to stdout instead of trying to send them:

```python
# In /home/janeway/janeway/src/core/settings.py
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

Then restarted the service:
```bash
sudo systemctl restart janeway
```

### What Console Backend Does
- All pages (submission confirmation, registration, etc.) display **normally** — no more Server Error
- Emails are written to the gunicorn console log instead of actually being sent
- **No real emails are delivered** — users won't receive inbox notifications
- Everything else works: submissions, reviews, registration, the full workflow

---

## How to Verify the Fix

1. Confirm the setting:
   ```bash
   grep -i "email_backend" /home/janeway/janeway/src/core/settings.py
   ```
   Should show: `EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"`

2. Test by making a submission at `https://heeropublishers.org/heero/submit/start/`
3. You should see a success/confirmation page — no Server Error

---

## How to Set Up Real Email (Future)

When you're ready to send actual emails, you need an SMTP provider. Options:

### Option 1: Gmail SMTP (Free, low volume)

```python
# In /home/janeway/janeway/src/core/settings.py
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.gmail.com"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = "your-email@gmail.com"
EMAIL_HOST_PASSWORD = "your-app-password"
DEFAULT_FROM_EMAIL = "Heero Publishers <your-email@gmail.com>"
```

**Important:** You need a Google App Password, not your regular password:
1. Go to https://myaccount.google.com/apppasswords
2. You need 2-Factor Authentication enabled first
3. Generate an app password for "Mail"
4. Use that 16-character password as `EMAIL_HOST_PASSWORD`

**Limits:** ~500 emails/day

### Option 2: SendGrid (Free tier: 100 emails/day)

```python
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.sendgrid.net"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = "apikey"
EMAIL_HOST_PASSWORD = "your-sendgrid-api-key"
DEFAULT_FROM_EMAIL = "Heero Publishers <noreply@heeropublishers.org>"
```

### Option 3: Mailgun (Free tier: 5,000 emails/month for 3 months)

```python
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.mailgun.org"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = "postmaster@mg.heeropublishers.org"
EMAIL_HOST_PASSWORD = "your-mailgun-password"
DEFAULT_FROM_EMAIL = "Heero Publishers <noreply@heeropublishers.org>"
```

### After Configuring Any Provider

1. Edit the settings file:
   ```bash
   sudo -u janeway nano /home/janeway/janeway/src/core/settings.py
   ```

2. Replace the console backend line with the SMTP settings from above

3. Restart:
   ```bash
   sudo systemctl restart janeway
   ```

4. Test by registering a new test account — you should receive the confirmation email

---

## Checking Logs

To see email-related output and errors:

```bash
# View recent service logs
sudo journalctl -u janeway --since "1 hour ago" --no-pager | tail -100

# Check service status
sudo systemctl status janeway -l --no-pager
```

**Note:** For journalctl to capture gunicorn output, the systemd service file needs:
```ini
[Service]
StandardOutput=journal
StandardError=journal
```

If these lines are missing, add them:
```bash
sudo nano /etc/systemd/system/janeway.service
sudo systemctl daemon-reload
sudo systemctl restart janeway
```

---

## Quick Reference

| Backend | Setting | Emails Sent? | Use Case |
|---------|---------|-------------|----------|
| Console | `console.EmailBackend` | No (logged to stdout) | Development / no SMTP yet |
| SMTP | `smtp.EmailBackend` | Yes | Production with real email |

**Current status:** Console backend (no emails being sent)
