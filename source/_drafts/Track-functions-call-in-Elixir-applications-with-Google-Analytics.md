---
title: Track functions call in Elixir applications with Google Analytics
tags:
  - elixir
published: true
---
Recently I decided to make public a telegram bot I did to monitor bus time in Dublin (TK link). Before the release I became curious to see how many people will use it (spoiler: just an handful) and I thought that would be nice to track the use on google analytics.

###Implementation

Google analytics provide a measurement protocol that can be used to track things that are different from websites (mobile apps, IOT). At the moment no elixir client exists for this protocol (and it would not be anything more than an api wrapper). My plan is to make call to the Ga TK endpoint with httpoison but I'd prefer to not have to call the tracking function for every single bot command. 
