+++
title = "On searching for pinouts"
date = 2025-04-20
[taxonomies]
tags = ["reverse-engineering"]
+++

Sometimes when you're trying to fix or piece together the inner workings of a piece of tech, you may stumble upon an IC that has neither datasheets nor even pinouts publicly available. Here's one more relatively simple way that sometimes helps me find the latter fellas.

## Idea

I utilize the fact that there are lots of leaked schematics publicly available, and they may contain the IC in question. For example, my backup of a telegram channel named [shematics&boardview laptop](https://t.me/schematicslaptop) I made a while back contains about 28GB of mostly pdf files, which you can search through with [pdfgrep](https://gitlab.com/pdfgrep/pdfgrep) or [ripgrep-all](https://github.com/phiresky/ripgrep-all).

## Obtaining schematics

The most convenient way to bulk-download stuff from telegram that I've found is through [tdl](https://github.com/iyear/tdl). Basically, you need to [login](https://docs.iyear.me/tdl/getting-started/quick-start/#login) into your account and export the channel in question (either with the desktop client, or via tdl, the docs on downloads explain it well), then start [downloading](https://docs.iyear.me/tdl/guide/download/), sit back and relax. 

## Searching

Is as easy as `rga -i "ps8339" -t pdf -l > ../ps8339.res`. This can also be extended into a telegram bot or something.
