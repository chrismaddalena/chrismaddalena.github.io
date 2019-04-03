---
layout: page
title: Projects
subtitle: Active FOSS projects
---

These are my active projects available on GitHub:

#### ODIN

https://github.com/chrismaddalena/ODIN

ODIN is my largest project to date. It has been in some phase of development since I presented it at SecTor in 2016. Today, ODIN is an automated asset discovery tool. It automates intelligence gathering by taking your input and performing dozens of searches and look-ups through search engines, APIs, and other tools to discover email addresses, IP addresses, subdomains, certificates, and more. ODIN tries to automate the more tedious tasks that are part of preparing for an engagement so the penetration tester can work more efficiently and focus on the analysis.

#### GhostManager

https://github.com/GhostManager

GhostManager is a collection of tools focused on reporting and infrastructure automation for red teams. This is a project created with contributions from other members of the SpecterOps team.

#### Goreport

https://github.com/chrismaddalena/Goreport

Goreport is a companion to Jordan Wright’s awesome GoPhish phishing platform and framework. GoReport uses GoPhish’s REST API to interact with your GoPhish server and generate statistics and full reports (CSV and Word documents) for your phishing campaigns.

#### Fox

https://github.com/chrismaddalena/Fox

Fox is a companion tool for BloodHound. It connects to your Neo4j project containing your BloodHound data, validates the data, and then generates statistics to assist with Active Directory analysis and hardening efforts. Fox runs queries that BloodHound can’t, i.e. queries that must be usually be run through Neo4j’s console. A penetration tester can run Fox on fresh BloodHound data during an assessment or during a post-mortem review for adversary resilience analysis. Defenders can also use Fox and BloodHound to perform regular reviews of their own networks.

#### Cooper

https://github.com/chrismaddalena/Cooper

Cooper is one of my earliest Python projects. I took it on as a challenge to make something more than a basic script. Cooper parses webpages and emails, clones them, and prepares them for phishing campaigns. It handles the tedious work of fetching webpage content, replacing relative links with full links to scripts and stylesheets, and decoding encoded email source. It then makes the resulting HTML human readable and ready for editing.