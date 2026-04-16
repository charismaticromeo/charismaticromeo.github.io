---
title: "Batch Renaming Like A Pro"
date: 2026-04-12
Categories:
  - Systems-Infractructure
  - Bash
tags:
  - automation
  - filesystem
---
## **Introduction**
On this blog, we are going to discuss about bulk renaming files with a uniform unwanted portion.

Pirates in the house, Hello!!! 

![Jack Sparrow Gif](/assets/images/100.webp)

Hope y'all feel seen.

If you frequent sites that offer "free" multimedia downloads, you already know that most of the times those files come with markers such as domain names from the site and so on. This in itself is an effort to self advertise, which is fair - It's not like they are gonna pay for TV prime time commercial. But those extensions look terrible on media libraries.

I must say that this post isn’t about promoting piracy, it’s rather about efficiency. Whether you’re cleaning up "acquired" media or professional data exports that came with weird timestamps, you don't have to rename files one by one like a maniac.

I do recognize that there are other means to conduct renames but only this one has the ability to leave you with hacker creds among your loser peers (^_^).

![Hacker Meme](/assets/images/hacker-meme.webp){: width="50%" }

## **Getting Started: Create a Pactrise Range**

Before you start hammering actual files, lets generate 40 dummy files to
practfice on.

```bash
for i in {001..040}; do touch "temp_file_$i.md"; done
```

This creates a clean stash of files names `temp_file_001.md` through
`temp_file_040.md`.

## 1. Stripping the prefix. 
The goal here is to remove the `temp_` portion of the filenames.

```bash
for file in temp_*; do mv $file ${file#temp_}; done
```
**Command Breakdown:**
- `for file in temp_*;` - This loop iterates over every file in the current directory whose name starts with *`temp_`* and stores the name in the variable name `file`.
- `do mv $file ${file#temp_};` - For each file `$file`, `mv` command renames it from the original*`temp_file_[number].md`* to a new name with prefix *`temp_`* removed.

## 2. Stripping the Suffix
Now, what if the unwanted bit is at the end? For example, your files are named `video_SITE_XYZ.mp4` and you want to remove `_SITE_XYZ`.

The Command:

```bash
 for file in *_SITE_XYZ.mp4; do mv "$file" "${file%_SITE_XYZ.mp4}.mp4"; done
```

`${file%pattern}`: The % symbol tells Bash to look at the end of the string, doing away with suffix.

## 3. The "Search and Replace" (The Middle)
Sometimes the garbage is stuck in the middle of the filename. Bash can handle that too using the `/` operator.

**The Command:**

```
Bash
for file in *; do mv "$file" "${file/UNWANTED_BIT/CLEAN}"; done
```

## Pro-Tips

**1. Dry Run for Safety:** Add an echo before the mv to see what would happen without actually changing anything i.e.,

```bash
for file in temp_*; do echo mv "$file" "${file#temp_}"; done
```

**2. Dealing with Spaces:** If your files have spaces (e.g., Movie Title [Site Name].mp4), the basic command will fail. Always wrap your variables in double quotes to keep the filename intact i.e.,

```bash
mv "$file" "${file#temp_}"
```

**3. Long vs. Short Matches**
> - A single # or % removes the shortest match.
> - A double ## or %% removes the longest match.

**Example:** *If your file is `temp_backup_temp_file.txt`, `${file##temp_}` would strip everything up to the `second temp_.`*

## Conclusion
You now have the power to clean up an entire hard drive of messy filenames in seconds. No extra software, no manual clicking. Just pure and efficient Bash logic.

