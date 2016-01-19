---
title: Project Conclusion of m.21dianping.com
date: 2016-01-20 01:13:39
tags: [project]
---

# About This Project  
The project is a commercial website where users share their experiences of renting houses.  
I accepted this outsourced project in late April and finished in about 15 days.  
The project required 2 developers including a front-end engineer and a back-end developer(me).

{% blockquote %}
 **Tasks**: development of the full site, deployment on server, maintainance for 3 months  
 **Profits**: 4500 RMB in total, 3000 for back-end developer & maintainer (me), 1500 for front-end engineer.
{% endblockquote %}

# Technical Info  

{% blockquote %}
**Back-end Language & Framework**: Python, Flask  
**Database**: MongoDB with mongoengine as ORM python engine  
**Front-end Frameworks**: Bootstrap(UI), Angular.JS(MVVM data-binding)  
**Front-back Interaction Method**: REST API  
**Login Method**: Only OAuth, logging in with Sina Weibo account or QQ account  
**Production Environment**: Nginx + uwsgi  
**About Server**: Ali Cloud, Ubuntu 14.04
{% endblockquote %}

# What I've Done
* Back-end development
* Significant modification of front-end code, including interaction with back-end using AJAX & REST API calls (The front-end developer was really a rookie so I had to do much front-end stuff to fill his void. In fact he did nothing but wrote some static HTML & CSS.)
* Server-side production environment set-up

# What I'm Doing
* Adding source code comments
* Working on maintainance documentation
* Website & server maintainance

# What I've learned
* Further understanding of Flask
* Bootstrap
* Front-end MVVM model using Angular.JS
* Production environment setup with nginx+uwsgi

# Improvement
Search algorithm in back-end source needs to be improved. Current solution of house-info searching is to traverse all records in database and calculate match-weight for each row. However when the number of records grows there may be significant performance issues.
