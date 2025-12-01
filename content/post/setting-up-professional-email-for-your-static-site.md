---
author: "Bírd Màn"
title: "Setting Up Professional Email for Your Static Site"
date: "2025-09-27"
description: "Setting Up Custom Domain Email."
summary: "Elevate your brand's credibility and strengthen your online presence with professional email accounts that match your custom domain for your static site."
tags: ["Technology", "Tools"]
categories: ["Engineering"]
aliases: ["setting-up-professional-email-for-your-static-site"]
thumbnail: "images/banner-setting-up-professional-email-for-your-static-site.webp"
---

In the [previous article](https://th3b1rdm2n.site/post/building-static-site-with-hugo-clarity-theme/), we deployed a static site. Now, let’s take it a step further by creating a branded professional email (e.g., connect@th1rdm2n.site) for the site. A custom email not only lets you send and receive messages under your own domain, but it also builds trust and credibility for your brand.

_Sign Up on Zoho Mail_  
Head over to [Zoho Mail](https://zoho.com/mail/?zmc=zoho) and with the **Business Email** button selected, 
- Enter your full name, a personal email address (e.g., username@gmail.com), and a password of your choice.
- Accept the Terms of Service and click Sign Up.
- Enter the OTP sent to the personal email you provided.

_Choose the Free Plan & Add Your Domain_  
By default, Zoho may prompt you toward paid plans. To switch:   
In the address bar, replace the URL with `https://mailadmin.zoho.com/hosting?plan=free`
- Click on **Add now** for the Add an existing domain card.
- Enter your domain (e.g., th1rdm2n.site) and your company name.
- Choose an `Industry Type`, then click **Add now**.

_Verify Your Domain Ownership_  
Zoho will ask you to verify your domain:
- Click **Proceed to domain verification**.
- Sign in to your domain registrar (e.g., Namecheap).
- Copy the TXT record Zoho provides.
- Paste it into your registrar’s DNS management panel.
Once added, return to Zoho and clic **Verify TXT Record** to confirm.

_Create Your Professional Email Address_
- Enter the email ID you want (e.g., connect@th1rdm2n.site).
- Click **Create**.
- Next click **Proceed to Setup Groups** - You can always setup group later.

_Configure DNS for Email Delivery_  
To ensure your branded email works reliably:
- Click **Proceed to DNS Mapping**.
- Copy Zoho’s MX, SPF, and DKIM records.
- Add them into your registrar’s DNS settings.
- Back on Zoho click **Verify all records** and then click **Proceed to Email Migration**.

_Complete Setup_
- Click **Proceed to Go Mobile**.
- Download the Zoho Mail app if desired.
- Finish by clicking **Proceed to Setup Completion**.
And that’s it — your professional email is live and ready to use.

For a step-by-step visual guide, check out the video
{{< youtube zbRd8P_1eO0 >}}
