---
author: "Bírd Màn"
title: "Integrating Analytics to Your Static Site"
date: "2025-10-03"
description: "Track and understand your visitors with Google Analytics."
summary: "Integrate Google Analytics into your Hugo-powered static site and start gathering insights into your audience’s behavior."
tags: ["Technology", "Engineering"]
categories: ["Research"]
aliases: ["integrating-analytics-to-your-static-site"]
thumbnail: "images/banner-integrating-analytics-to-your-static-site.webp"
---

In our [last article](https://th3b1rdm2n.site/post/setting-up-professional-email-for-your-static-site/), we added a custom domain email to build brand credibility. Now, let’s make your site smarter by adding **Google Analytics** — a free and powerful tool that helps you understand your visitors - who’s visiting, where they’re coming from, and what they’re doing.

#### Step 1: Create a Google Analytics Property

Head over to [Google Analytics](https://analytics.google.com/):

- Sign in with your Google account.
- Click **Start Measuring**.
- Enter an **Account Name** (e.g., `eyerie`).
- Click **Next**, then give your **Property** a name (e.g., `th3b1rdm2n-site`).
- Choose your **Time Zone** and **Currency**, then click **Next**.
- Choose your business size and industry, then click **Next**.
- Check the boxes that aligns with your business objective, then click **Create**.
- Check the **Google Analytics Terms of Service Agreement** and click **I Accept**.

---

#### Step 2: Get Your Measurement ID

You’ll now be on the **Web Stream Details** page:

- Click **Web** as the platform.
- Enter your **site URL** (e.g., `th3b1rdm2n.site`) and a stream name(e.g, `th3b1rdm2n-site` ).
- Click **Create & continue**.
- Click **Next** and copy the Measurement ID (starts with `G-`).

We’ll use this ID to connect your Hugo site.

---

#### Step 3: Add the Tracking Code to Hugo

In your Hugo site’s root directory:

1. Open the `params.toml` (located at `config/_default`).
2. Add the following under `google_tag_manager_id` (if using Clarity or a similar theme):

```toml
# Google tag manager
google_tag_manager_id = "G-XXXXXXXXXX"
```

---

#### Step 4: Rebuild and Deploy

After updating the config file, rebuild your site and push the changes to GitHub Pages:

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

#### You're All Set 🎉

Now that Google Analytics is integrated, your Hugo site can give you insights into:

- Visitor locations  
- Page popularity  
- Traffic sources  
- Device types  

It’s a small step for your site, but a giant leap for understanding your audience.

For the full walkthrough, here’s a companion video:  
{{< youtube uR8SzMnxLB0 >}}
