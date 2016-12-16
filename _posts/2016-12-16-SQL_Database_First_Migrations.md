---
layout: post
title: SQL Database First Migrations
published: false
comments: true
---

Well, as an opening post... I would have probably never started if I would wait for some dream-topic to describe. 

# What's the problem
Probably database-first is still a frequent way of approaching databases. I had encountered some projects in my life that had some manual-driven, handy work done to make an update. One project (that was still before it's production stage) had something like a "change-backup-restore" cycle to make the changes. It was like this:
- somebody wanted to make a change on a database - he shouted to the team, that he will be doing so (yep, that is a "lock" on the database mechanism)
- the person firstly restored a last backup of the database (so the changes made were clean), done his changes...
- ...and made a backup which was then commited to SVN
- the person said, he has released the "lock".
- everybody downloaded the backup and restored on their version.

It would be probably easier to have at least a shared Dev database, right?. The same was applied to the UAT environment, because there was no production at that time. I've came in to the project after a few months of this madness and... the client wanted to go Live! 

Before we have gone to a state of automatic updates, there was some time of adding scripts to source control, which the "deployer" will execute one by one - manually. With that approach we were safe, but loaded with a lot of manual work and a little pinch of stress on deployments. It had to stop.

# What's the idea
What ideas developers have for solving this? 
- the one I've described - manual scripts ( bad idea)
- SQL compare tools (like Redgate SQL Compare - http://www.red-gate.com/products/sql-development/sql-compare/ ) - not bad, but still some manual work. What about some test stuff developers insert in a database? What if you need to make an explicit insert to the production?
- How about ... database-first migrations?

Code-First approach has this cherry at the top - Migrations. Run Add-Migration after defining your class, and voila - you get a fully-automatic migration class with Up and Down methods so you can move anywhere in the migration flow. The concept of a migration is that you divide your changes into iterations (even the initial DB version) and execute them one by one as your project grows and changes.

Why not apply this approach to Database-First? The migration can be a SQL script, which will also have a number and will be executed in order. Making an Up/Down method isn't trivial, but having a Up script that is executed in an automatic manner is already a big jump forward. 

# What's the solution
TODO
