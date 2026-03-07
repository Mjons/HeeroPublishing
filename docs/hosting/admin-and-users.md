# Admin & User Management

## Logging In

- **Press Manager**: `https://heeropublishers.org/manager/`
- **Django Admin** (full database access): `https://heeropublishers.org/admin/`
- **Journal Manager**: `https://heeropublishers.org/heero/manager/`

---

## User Roles

| Role | What they can do |
|------|-----------------|
| **Superuser** | Everything. Full Django admin access. Server-level control. |
| **Staff** | Access Django admin panel (limited by permissions) |
| **Editor** | Manage submissions, reviews, copyediting for their journal |
| **Journal Manager** | Journal settings, users, and content for their journal |
| **Author** | Submit articles |
| **Reviewer** | Review assigned articles |
| **Reader** | Browse published content |

---

## Creating a New User

### Option A: Through Press Manager (recommended)

1. Go to `https://heeropublishers.org/manager/`
2. Click **All Users (New)**
3. Fill in email, first name, last name
4. Set a temporary password
5. Assign roles as needed

### Option B: Through Django Admin

1. Go to `https://heeropublishers.org/admin/`
2. Click **Accounts** under CORE
3. Click **Add Account** (top right)
4. Fill in email and password
5. Check **Staff status** if they need Django admin access
6. Check **Superuser status** if they need full control

### Option C: From the command line (on server)

```bash
su - janeway
cd janeway && source venv/bin/activate
JANEWAY_SETTINGS_MODULE=core.settings python src/manage.py createsuperuser
```

---

## Making Someone a Superuser

If the user already exists:

1. Go to `https://heeropublishers.org/admin/`
2. Click **Accounts** under CORE
3. Find the user (search by email)
4. Click their email to edit
5. Scroll down and check:
   - **Staff status** = checked
   - **Superuser status** = checked
6. Click **Save**

---

## Setting Up the Client's Account

1. Get their email address
2. Create their account (Option A or B above)
3. Make them a Superuser (so they have full control of their press)
4. Give them their login credentials
5. Both you AND the client should be superusers

---

## Resetting a Password

### Through Django Admin
1. Go to `/admin/` → **Accounts** → find user → click **Change password** link

### From command line
```bash
su - janeway
cd janeway && source venv/bin/activate
JANEWAY_SETTINGS_MODULE=core.settings python src/manage.py changepassword user@email.com
```

---

## Important Notes

- The admin email set during `install_janeway` is the primary superuser
- Multiple users can be superusers — no limit
- Journal-level roles (editor, reviewer) are assigned per journal
- Press-level roles (press manager) control the whole platform
