---
title: "Bulk File Renaming with Bash Expansions"
date: 2026-04-12
Categories:
  - Systems-Infractructure
  - Bash
tags:
  - automation
  - filesystem
---
On this blog post, we are going to discuss about bulk renaming files with a uniform unwanted bit.

Pirates in the house, Hello!!! 

![Jack Sparrow Gif](/assets/images/100.webp)

Hope y'all feel seen.

If you frequent sites that offer "free" multimedia downloads, you would already know that most of the times those files come with markers such as domain names for the site etc. I see it as an effort to self advertise, which isn't a bad thing. It's not like they are gonna pay for a TV prime time commercial someday.

This article isn't focused on promoting piracy either, I am using this example because it's the most relatable case. It doesn't mean that material acquired on 'legal' mean don't contain markers. You could end up with funny \*fixes for any reason on your filenames, and my job here to make sure you don't go on renaming every file one by one like a maniac.

I do recognize that there are other means to conduct renames but only this one has the ability to leave you with hacker creds among your loser peers (^_^).

Let's get started.

**1. Generating a stash of files to practice on**

```bash
for i in {001..40}; do touch "temp_file_$i.md"; done
```

```bash
for file in temp_*; do mv $file ${file#temp_}; done
```
**Command Breakdown:**
- `for file in temp_*;` - This loop iterates over every file in the current directory whose name starts with *`temp_`* and stores fthe name in the variable name file for each iteration.
- `do mv $file ${file#temp_};` - For each file `mv` command renames it from the original name in *`$file`* to a new name with prefix *`temp_`* removed.
