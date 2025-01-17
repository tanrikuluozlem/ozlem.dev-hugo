+++ 
draft = false
date = "2025-01-17T00:00:00+03:00"
title = "How I Used Git Rebase to Clean Up and Refine My Commit History"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

While working on a GitHub repository where I'm building the backend, I noticed that my commit history needed some restructuring.

Some commit messages were clear, while others were inconsistent or even a bit playful like "read only Friday :)". At the time, it felt cool to include something fun in my history. But as I thought about making my repository open for collaboration, I realized that this wasn't the best way to present my work.

I wanted a clean, readable commit history that told the story of my project without any distractions. To do that, I turned to Git rebase. In this post, I'll share how I used it to organize my commit history step by step.

Why Did I Care About My Commit History?

Your commit history is more than just a log of changes - it's part of the identity of your project. If you've ever opened a repository to find a series of unclear or poorly written commit messages, you know how it can impact the project's clarity.

Using Git rebase, I was able to rewrite my commit history into something clear, logical, and professional.

Here's what I wanted to achieve:

* To make commit messages meaningful: Descriptive messages help everyone understand the history of changes. So update "read only Friday :)"
* To correct mistakes: Typos like "chunksCahce " (instead of chunksCache) made the history harder to read.
* To merge redundant commits: Small, unrelated commits clutter the history unnecessarily.

To edit my commit history, I used interactive rebase. First, I opened the last six commits for review:

{{< highlight bash >}}

git rebase -i HEAD~6

{{</ highlight >}}

This command brought up an editor showing my recent commits:

{{< highlight >}}

pick 43d5cee perf(neu-web-test): Scale down wp-test hpa min replicas
pick 88ccfc2 feat(loki): Add chunksCache and resultsCache resources 
pick 825ca0d Fix typo in adding chunksCahce logic
pick a9f12b3 increase cpu requests 
pick bd9e4a8 Update README
pick 05d3f61 read only Friday :)

{{</ highlight >}}

In the rebase editor, I updated the commands to specify what I wanted to do with each commit:

{{</ highlight >}}

pick 43d5cee perf(neu-web-test): Scale down wp-test hpa min replicas
pick 88ccfc2 feat(loki): Add chunksCache and resultsCache resources 
fixup 825ca0d Fix typo in adding chunksCahce logic
pick a9f12b3 increase cpu requests 
reword bd9e4a8 Update README
reword 05d3f61 read only Friday :)

{{< highlight >}}

Here's what these commands mean:

* pick: Keep the commit as-is.
* fixup: Combine the commit with the one above it, discarding its message.
* reword: Edit the commit message.

Edit Commit Messages

After saving the file, Git guided me through the changes step by step.

For the commit "read only Friday :)", I rewrote the message to something clearer:

{{</ highlight >}}

Restrict editing to Fridays

{{</ highlight >}}

Update README with setup instructions

{{</ highlight >}}

Update README with setup instructions

{{</ highlight >}}

Once the rebase was complete, I checked the updated commit history:

{{</ highlight bash >}}

git log --oneline

{{</ highlight >}}

The new history looked like this:

{{</ highlight >}}

43d5cee perf(neu-web-test): Scale down wp-test hpa min replicas
88ccfc2 feat(loki): Add chunksCache and resultsCache resources 
a9f12b3 increase cpu requests 
bd9e4a8 Update README with setup instructions
05d3f61 Restrict editing to Fridays

{{</ highlight >}}

Organizing my commit history with Git rebase wasn't just about the technical details it was about presenting my work in the best light possible. Now, I feel confident sharing my repository with collaborators, knowing it's clear and easy to navigate.

If you've been avoiding Git rebase, I'd say give it a try. It's simpler than you might think, and it really helps keep your commits clean.




