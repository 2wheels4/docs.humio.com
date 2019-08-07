---
title: "Sharing Dashboards"
aliases: ["/shared-dashboards", "/wall-monitors", "/ref/dashboards/sharing"]
menuTitle: Wall Monitor Dashboards
---

If you want to put a dashboard up on a wall monitor or grant read-only access to individual widgets or entire dashboard
publically or to a limited group you can use Humio's "Shared Secret URLs" feature.

A shared secret URL is a URL containing a special authentication token that grants read-only
access to anyone that has the link.

## Creating a Shared Secret URL

1. Visit the dashboard you want to make a read-only link for

2. Click the "More" menu item (icon with three dots).

![The more menu](/images/pages/dashboards/more-icon.png)

3. Click "Wall Monitors & Shared URLs"

![Accessing the shared links menu](/images/pages/dashboards/wall-monitors.png)

4. Give the link a name and click "Create Link"

![Creating a shared link](/images/pages/dashboards/links.png)

5. Visit the shared URL by clicking the Link in the URL column.

You can now use this link in the browser of your wall monitor or send it to people
that should have read-only access.

## FAQ

### Why not just make a seperate user for wall monitors?

Humio's security model will force a user to re-authenticate after the session expires,
since wall monitors are usually non-interactive (don't have a keyboard) - if you do it
this way you will need to figure out how to make the browser re-authenticate periodically.

### Are shared secret URLs safe

They are as safe as any shared secret, but if anyone has the URL, they have read-only access.
This might not be acceptable for your organization.

In any case there are audit logging and GDPR to considerations you need to make.
Often you need to know which users had access to what and when.

Under any circumstance we recommend that you limit access to the Humio machines with a
firewall or similar, to limit the impact of URLs getting into the wrong hands.  
