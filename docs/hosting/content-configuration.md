# Content & Branding Configuration

Everything below is done through the browser — no coding needed.

---

## 1. Press Branding (the main site)

Go to `https://heeropublishers.org/manager/` → **Edit Press Details**

| Setting | What it does |
|---------|-------------|
| Press name | Shown in header and title |
| Logo | Replaces the default Janeway logo |
| Footer text | Custom footer content |
| Favicon | Browser tab icon |
| Description | About text for the press |

---

## 2. Press Homepage Elements

Go to `https://heeropublishers.org/manager/` → **Homepage Elements**

These are the blocks that appear on the main press homepage. Available elements:

| Element | What it shows |
|---------|--------------|
| **Carousel** | Image slider at the top |
| **Journals** | List of all journals |
| **Journals and HTML** | Journals list + custom text |
| **News** | Latest news items |
| **HTML** | Custom HTML/rich text block |
| **Preprints** | Link to preprint repository |
| **Search Bar** | Search widget |

### To enable an element:
1. Click **Homepage Elements**
2. Find the element you want
3. Toggle it on
4. Drag to reorder (order matters — top = first on page)
5. Click the element name to configure it (e.g., carousel images)

---

## 3. Journal Configuration

Go to `https://heeropublishers.org/heero/manager/`

(Or from Press Manager, click on the journal name)

### Journal Settings (Edit Settings)
- Journal name and description
- ISSN number
- Journal logo and header image
- Contact information
- Copyright and license defaults

### Journal Homepage Elements
Same as press but for the journal page (`/heero/`):

| Element | What it shows |
|---------|--------------|
| **About** | "About this journal" section |
| **Carousel** | Image slider |
| **Featured Articles** | Highlighted articles |
| **Current Issue** | Latest published issue |
| **News** | Journal news |
| **Popular Articles** | Most viewed articles |
| **HTML** | Custom content block |
| **Search Bar** | Search widget |

---

## 4. Pages (CMS)

Go to Press Manager → **Content Manager** (or `/cms/`)

Create custom pages like:
- About Us
- Editorial Board
- Submission Guidelines
- Contact
- Policies

Each page gets its own URL and can be added to the navigation menu.

---

## 5. Navigation Menu

Go to Press Manager → **Content Manager** → **Navigation**

Add/remove/reorder links in the site navigation bar.

---

## 6. News Items

Go to Press Manager → **News Manager**

Add announcements, updates, calls for papers, etc. These show up on the homepage if the News homepage element is enabled.

---

## 7. Themes

Janeway has 3 built-in themes:
- **OLH** (default) — Clean academic look
- **Material** — Modern Material Design style
- **Clean** — Minimalist

To change: Go to `/admin/` → **Press** → click your press → change **Theme** field.

After changing theme, restart gunicorn on the server:
```bash
ssh root@67.207.94.240
systemctl restart janeway
```

---

## Quick Setup Checklist (minimum to look professional)

- [ ] Upload press logo (Edit Press Details)
- [ ] Set press description (Edit Press Details)
- [ ] Enable "Journals" homepage element (Homepage Elements)
- [ ] Enable "About" or "HTML" homepage element with welcome text
- [ ] Set journal name and description (Journal → Edit Settings)
- [ ] Upload journal logo (Journal → Edit Settings)
- [ ] Set ISSN if available (Journal → Edit Settings)
- [ ] Create an "About" page (Content Manager)
- [ ] Create an "Editorial Board" page (Content Manager)
- [ ] Add pages to navigation (Content Manager → Navigation)

---

## SMTP Email Setup (needed before users can register/reset passwords)

Edit `src/core/settings.py` on the server and add:

```python
EMAIL_HOST = "smtp.your-provider.com"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = "your-smtp-user"
EMAIL_HOST_PASSWORD = "your-smtp-password"
DEFAULT_FROM_EMAIL = "noreply@heeropublishers.org"
```

Free SMTP options:
- **Brevo** (free: 300 emails/day)
- **Gmail SMTP** (free: 500 emails/day, needs app password)
- **SendGrid** (free: 100 emails/day)

After editing, restart gunicorn:
```bash
systemctl restart janeway
```
