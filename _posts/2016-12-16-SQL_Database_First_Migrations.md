---
layout: post
title: SQL Database First Migrations
published: false
---

Well, as an opening post... I would have probably never started if I would wait for some dream-topic to describe. 

# What's the problem
Probably database-first is still a frequent way of approaching databases. I had encountered some projects in my life that had some manual-driven, handy work done to make an update. One project (that was still before it's production stage) had something like a "change-backup-restore" cycle to make the changes. It was like this:
- somebody wanted to make a change on a database - he shouted to the team, that he will be doing so (yep, that is a "lock" on the database mechanism)
- the person firstly restored a last backup of the database (so the changes made were clean), done his changes...
- ...and made a backup which was then commited to SVN
- the person said, he has released the "lock".
- everybody downloaded the backup and restored on their version.

Maan, it would be probably easier to even have a shared Dev database. The same was applied to the UAT environment, because there was no production at that time. I've came in to the project after a few months of this madness, 
