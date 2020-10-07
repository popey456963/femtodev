+++
layout = "post"
title = "Architecture of Minimal Design"
date = "2020-06-21"
categories = ["Bash", "Linux", "Node"]
draft = true
+++

Minimal design is the name given to a monorepo of websites that all share the same authentication and web serving systems.  It was originally the host of systems like https://femto.pw and others, which have now been moved to a newer "Femto Apps" architecture.  There are several key tenets to the minimal design system:

- Webpages should be small
- Webpages should work without JavaScript
- Webpages should be mobile repsonsive
- Creating a webpage should only take a few minutes

I was in a phase where I created a large number of small tools which usually only required user authentication to save data between visits.
