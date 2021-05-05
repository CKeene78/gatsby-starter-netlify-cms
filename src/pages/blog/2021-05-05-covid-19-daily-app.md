---
templateKey: blog-post
title: COVID-19 Daily App
date: 2021-05-05T19:41:00.000Z
description: >-
  Like many places that are trying to bring people back to work our facility
  wanted to implement a program for individuals, specifically staff, to do daily
  self-assertion that they are not sick and have not engaged in risky behavior.
  Has since become a way to remain within industry compliance requirements.  
featuredpost: true
featuredimage: /img/Screenshot 2021-05-05 at 3.44.56 PM.png
tags:
  - HTML
  - JavaScript
  - CSS
  - Vue
  - Web App
---
# Links

Demo: [https://covid19selfcertify.keenecreations.com](https://covid19selfcertify.keenecreations.com/)

GitHub: <https://github.com/CKeene78/COVID19SelfCertify>

# Solution

In order to generate data you need a way to collect it first. So,what are users going to see and interact with? A web page, of course! I’m a big fan of single page applications (SPA) and the questions and flows are rather simple. So we have a single HTML file divided into several `<section>` tags that represent individual screens. Only one `<section> `will be seen at a time as determined by the section’s `visible` property, and each `<section> `is absolutely positioned so will always show up at the top of the screen. Essentially each `<section>` is stacked on top of each other and we use JavaScript and CSS to determine which *page* is shown.We use Vue to collect the responses across the screens and send to the middle processing tier.

# Technologies

## Visual Studio Code (VSCode)

Programming IDE tool of choice that is free, fast and almost infinitely extensible.

## SharePoint

SharePoint Lists are adequate for our purposes here as we only have a few metrics we are capturing for a relatively small population.Filtering and examining the data is also rather easy in SharePoint Lists especially with Power BI queries.The back end and included in our Office 365 Enterprise licenses.

## Azure Static Website + GitHub

Hosts the front end website.We opted to connect the Static Website to a subdomain rather than the default URI that Azure provides.Connected to a GitHub repository and are rebuilt each time the repository is updated so users will always be using the latest version of the program.Free!

## HTML/JS/CSS + Vue

The front end that users will interact with, collect the answers, and interacts with the processing tier. Free!

## Power Automate

Our processing tier that communicates between the external front end and internal back end. There are several flows that triggered by a POST to REST endpoints.Example flows are: informing the user if they should come into work or stay home; recording the form answers; and automatically emailing alerts to staff nurse and a person’s supervisor. Included in our Office 365 Enterprise license.

# Result

Whew! So what is the result of all of that? Was it worth it? We have estimated that by developing an in-house application utilizing the Power Platform and Office 365 Enterprise saves us the $4,000 - $10,000 per year over the COTS vendor that we would have chosen otherwise. Other vendors were even more expensive than that! Great return on investment for our company. Sadly I have yet to see any of that money in my paycheck.

# What about COTS?

There are many software companies that are beginning to provide commercial off the shelf  services on a monthly per-person subscription basis. Affordable, typically tiered, provide numerous reporting and tracking option via an app and/or web site. While investigating these options we quickly realized that we did not currently, nor planned to, require the majority of the functionality of the programs. Very inexpensive per person but, depending on how many people need to be covered, can quickly balloon to a non-trivial monthly expense. Most vendors also charge on a per change basis beyond their canned questions, which can also add up quickly as people begin to use the program. Let’s see what it will take to be more flexible and save some much-needed funds.
