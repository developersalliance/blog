---
layout: post
title: "Hacking Attempt and Admin credential leak."
author: Giga
categories: [Magento, Hacking, Debugging, patch]
image: assets/images/blog/Magento-hacked-image-2.png
featured: true
hidden: true
---

Magento 2 Checkout Bug and CMS Block Attack
===========================================

The Bug
-------

Last week, one of our automated browser tests sent us a red alert. Something was wrong with the checkout on a Magento 2 site. It couldn't fill out the input fields and complete the checkout anymore, and that's a big deal when you're running an online store.

At first, we thought it might be some regular bug. But when I opened the browser console, I saw something odd happening on every page, there was a debugger loop being injected. 

![](https://lh7-rt.googleusercontent.com/docsz/AD_4nXegHCRIvx4QHTc5LUt517rSs1QI7t__BrRhlJGorfTJUXpnrbhEgnBKNzCt7P6-twKHyLtoUNM3b2MSvNosA9OPPpYfig5xYmL0q1qh3_FwC5cE2F59KM8m8npdg9v5kA1fhHGNCg?key=qBeBNiy_AQmL92yr44iRyJcq)

This wasn't just a broken checkout. It was something more serious.

Digging Deeper
--------------

We followed our security checklist. First, we checked the server permissions and the access logs. Thankfully, no one had accessed the server directly, that would've been much worse.

Next, we checked the CMS Blocks in the Magento admin panel. That's when we found something really strange: machine-generated code injected into every block.

![](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcQ9Zv1zahSXZCWTyA5bPjS3Z_wR9xhPDg9nfp6KABYo9r_LlYxt7mdeq5UWCRBCOm-5ri_nJIkl1Iwkh-scUZ0df5L3cWAJ38xVbDwSw3-ANMvSFHj3X6CE0pL5eU_XnEeEpFT?key=qBeBNiy_AQmL92yr44iRyJcq)

I tried removing the code manually. It came back. I removed it again... and it came back again.

What is the damage?
-------------------

While inspecting the issue, the first question that came to mind was: *How serious is this?*

Did it leak any customer information? Did it collect any data? Did it change any other site content?

After checking the code more closely, we saw that it wasn't running properly, thanks to our CSP rules and partly due to Cloudflare's WAF protection.

So, the only real damage was that the checkout stopped working, which we caught almost right away.

Could It Be the Patch?
----------------------

We realized a new Magento security patch had just been released the day before. Maybe it was related? We installed the patch and deployed it.

Still, the machine code kept reappearing. My first guess was that one infected block was spreading to others.

We had a backup of the database. Using Navicat, we exported all the CMS blocks, including block_websiteso everything stayed connected to the right store views.

We restored the clean blocks on production.

The Real Problem
----------------

A few minutes later, the blocks started getting corrupted again, one by one.

That confirmed our suspicion: someone still had access through an admin user account. Most likely, one of the login credentials had been leaked.

So, we immediately deactivated all admin users and refreshed the CMS blocks again.

This time, it worked. The problem was gone.

To be sure we scanned all the blocks for injected code, and checked template files. Everything looks good.

What We Learned
---------------

We've now tightened our security protocols even more. We strongly recommend two-factor authentication (2FA) or SSO for all admin users and Oauth authorization for all integrations.

We've also updated our internal policies by introducing an improved debug guide and agreeing with all of our clients that urgent patches should be installed immediately, no more waiting for the usual approval process.

Next up: we're building an automatic patch notification bot. Because staying secure shouldn't depend on a to-do list.

If you're running a Magento store and want to stay protected, make sure you're keeping your software up to date, enabling 2FA, and control and review admin access regularly. Or just reach out to us, we'll help keep your shop safe.
