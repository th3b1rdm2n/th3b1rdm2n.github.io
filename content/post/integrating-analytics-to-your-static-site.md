---
author: "Bírd Màn"
title: "Integrating Analytics to Your Static Site"
date: "2025-10-03"
description: "Track and understand your visitors with Google Analytics."
summary: "Learn how to easily integrate Google Analytics into your Hugo-powered static site and start gathering insights into your audience’s behavior."
tags: ["Technology", "Engineering"]
categories: ["Research"]
aliases: ["integrating-analytics-to-your-static-site"]
thumbnail: "images/banner-integrating-analytics-to-your-static-site.webp"
---

In our [last article](https://th3b1rdm2n.site/post/setting-up-professional-email-for-your-static-site/), we added a custom domain email to build brand credibility. Now, let’s make your site smarter by adding **Google Analytics** — a free and powerful tool that helps you understand your visitors.

> Know who’s visiting, where they’re coming from, and what they’re doing — all without adding bloat to your static site.

---

#### Step 1: Create a Google Analytics Property

Head over to [Google Analytics](https://analytics.google.com/):

- Sign in with your Google account.
- Click **Start Measuring**.
- Enter an **Account Name** (e.g., `eyerie`).
- Click **Next**, then give your **Property** a name (e.g., `th3b1rdm2n.site`).
- Choose your **Time Zone** and **Currency**, then click **Next**.
- Choose your business size and intent, then click **Create**.

> Accept the terms, and your new Analytics property will be ready.

---

#### Step 2: Get Your Measurement ID

You’ll now be on the **Web Stream Details** page:

- Click **Web** as the platform.
- Enter your **site URL** (e.g., `https://th3b1rdm2n.site`) and a stream name.
- Click **Create Stream**.
- Copy the **Measurement ID** (starts with `G-`).

We’ll use this ID to connect your Hugo site.

---

#### Step 3: Add the Tracking Code to Hugo

In your Hugo site’s root directory:

1. Open the `params.toml` (located at `config/_default`).
2. Add the following under `google_tag_manager_id` (if using Clarity or a similar theme):

```toml
# config/_default/params.toml
google_tag_manager_id = "G-XXXXXXXXXX"
```

---

#### Step 4: (Optional) Manually Insert GA Script

If your theme doesn’t support `googleAnalytics` out of the box, you can manually insert the tracking code.

Go to the Head section of your theme — usually in `layouts/partials/head.html`.

Paste the GA script just before the closing `</head>` tag:

```html
<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-XXXXXXXXXX');
</script>
```

---

#### Step 5: Rebuild and Deploy

After updating the config file, rebuild your site Push the changes to GitHub Pages:

```bash
git add -u
git commit -m "chore: added google analytics to site - $(date +%F_%H:%M)"
git push -u
git switch production
git merge development
git push -u 
git switch development
```

---

#### Step 6: Verify It's Working

Visit your site in a browser.

Go back to Google Analytics → Admin → Realtime.

You should see active users show up almost instantly.

---

#### You're All Set 🎉

Now that Google Analytics is integrated, your Hugo site can give you insights into:

- Visitor locations  
- Page popularity  
- Traffic sources  
- Device types  

It’s a small step for your site, but a giant leap for understanding your audience.

Stay tuned for more ways to level up your static site game. Until next time!

---

For the full walkthrough, here’s a companion video:  
{{< youtube Yk1yKzEp3C4 >}}
