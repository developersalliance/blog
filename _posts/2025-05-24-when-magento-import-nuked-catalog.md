---
layout: post
title: "When a Magento Import Almost Nuked a Product Catalog"
date: 2025-05-24
author: David
tags: [magento, import, patch]
featured: true
image: assets/images/nuke.jpg
---

I got the message on a regular weekday: "We lost almost the entire product catalog, around 10,000 products gone, only 100 left." The client had no idea what happened. They suspected something went wrong after running a product import, but the CSV file in question looked harmless, it had just a few items. It shouldn't have caused any damage. Yet, the damage was done.

Now the priority was restoration.

Normally, this kind of recovery is straightforward, since most clients have a centralized source of truth, like an ERP system, which can push the full catalog through a reimport. Not this client. Their catalog was built through layers of decentralized updates: ERP pushing new products via Magento's bulk API, manual updates through CSV imports, and one-off edits through the admin UI. There was no single reliable point to roll from.

Thankfully, the hosting provider had daily database snapshots. I pulled the snapshot from the day before. A full DB rollback wasn't an option, since we'd lose all the day's orders, customer data, and other updates. So I selectively restored the tables related to products. No unnecessary overwrites, no data loss. The site came back to life.

I also considered putting the snapshot's database on a local instance of the project, creating a CSV file using Magento's default exporter, and importing that CSV file to production. However, that would result in different `entity_id` and `row_id` values for each product, which would have caused problems in some cases like linking with sales data and other relational records that use those columns as foreign keys.

Then came the post-mortem.

The import history in Magento told an odd story. The most recent processed file was clean and nothing in it could have triggered this. But there was another file, one that had only been **validated**, not **imported**. This file *did* contain enough data to wipe out the catalog. But Magento shouldn't have processed a file tha was only validated, right?

Wrong.

As of Magento 2.4.6, a bug was introduced in the import mechanism. When a file is uploaded and validated, Magento writes its contents to the `importexport_importdata` table. If the merchant then uploads a new file and imports it, Magento doesn't discard the previously validated (but unprocessed) data, it merges it with the new one. Even worse, if the records of that file contain errors blocking it from being processed, the data lingers in that table indefinitely, unless someone manually clears it.

In our case, the client uploaded a destructive file, one that would've removed most of their catalog. They clicked "Validate" and realized it was wrong. So they corrected it and uploaded a proper version. But Magento, still clinging to the previous data in `importexport_importdata`, silently merged the two. The result: mass deletion of products.

Well, you migth be thinking that it's not that big of a deal, because you still have to upload such file in first place to cause this issue.

Which is right, but this bug introduces all kinds of confusion. Imagine uploading a corrected CSV, only to see Magento throwing errors about line numbers that don't exist in your file. That's the orphaned data still polluting the import table. 
So, this bug can cause problems in various shapes and forms.

The issue was patched in Magento 2.4.8. For anyone running 2.4.6 or 2.4.7, apply this patch immediately: [Issue #37593 : Fix Import Merge Bug](https://github.com/magento/magento2/issues/37593#issuecomment-2047590005)
