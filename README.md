# Deployra.ai — Deploy AI Agents to Production in 60 Seconds

> The managed deployment platform for AI agents. One CLI command. Any framework. Full observability.

## 🚀 Quick Deploy to GitHub Pages

### Step 1: Create GitHub Repository

```bash
# Go to github.com/new and create a repo called "deployra.ai"
# Make it PUBLIC (required for free GitHub Pages)
```

### Step 2: Push This Code

```bash
git init
git add .
git commit -m "Initial launch: Deployra.ai landing page"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/deployra.ai.git
git push -u origin main
```

### Step 3: Enable GitHub Pages

1. Go to your repo → **Settings** → **Pages**
2. Under **Source**, select **Deploy from a branch**
3. Select **main** branch and **/ (root)** folder
4. Click **Save**
5. Wait 1-2 minutes — your site will be live at `https://YOUR_USERNAME.github.io/deployra.ai`

### Step 4: Connect Custom Domain (deployra.ai)

**In GitHub:**
1. Go to repo → **Settings** → **Pages** → **Custom domain**
2. Enter `deployra.ai` and click **Save**
3. Check **Enforce HTTPS** (may take a few minutes to activate)

**In BigRock DNS Settings:**
Add these DNS records to point deployra.ai to GitHub Pages:

| Type  | Name | Value |
|-------|------|-------|
| A     | @    | 185.199.108.153 |
| A     | @    | 185.199.109.153 |
| A     | @    | 185.199.110.153 |
| A     | @    | 185.199.111.153 |
| CNAME | www  | YOUR_USERNAME.github.io |

> **Note:** DNS propagation takes 15 minutes to 48 hours. The site will work immediately at the github.io URL.

### Step 5: Verify

- Visit `https://deployra.ai` — should show your landing page
- Test the email form works
- Check mobile responsiveness

## 📧 Connecting Waitlist Form to Real Email Collection

Replace the `submitForm()` function in `index.html` with one of these:

### Option A: Google Forms (Free, Quickest)
1. Create a Google Form with one "Email" field
2. Get the form action URL and field name
3. Replace the console.log with a fetch to the form

### Option B: Buttondown (Free tier, 100 subscribers)
```javascript
fetch('https://buttondown.email/api/emails/embed-subscribe/YOUR_NEWSLETTER', {
  method: 'POST',
  body: JSON.stringify({ email: emailInput.value }),
  headers: { 'Content-Type': 'application/json' }
});
```

### Option C: Formspree (Free, 50 submissions/month)
```javascript
fetch('https://formspree.io/f/YOUR_FORM_ID', {
  method: 'POST',
  body: JSON.stringify({ email: emailInput.value }),
  headers: { 'Content-Type': 'application/json' }
});
```

## 📁 File Structure

```
deployra.ai/
├── index.html      # Complete landing page (single file, no dependencies)
├── CNAME           # Custom domain config for GitHub Pages
└── README.md       # This file
```

## Built with ❤️ by Deployra, Inc.
