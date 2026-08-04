# Day 02
- **Challenge name:** [Room 404]([https://tryhackme.com/room/hh-theconciergeknows-2d7eb4d9](https://tryhackme.com/room/hh-room404-804573bf))
- **Category:** Web Exploitation / Source Code Exposure
- **Difficulty:** Easy
- **Date completed:** 28th July 2026

## Summary
Following directly on the heels of Day 01, in this case, the Byte Lotus team rushed their guest-experience platform into a live production environment, cutting corners that resulted in a critical administrative oversight: an entirely unprotected .git directory left sitting on the web server. While the primary objective on paper was deceptively simple—dump the exposed source code and extract the flag—the challenge serves as a textbook exploration of poor artifact management during deployment. It highlights the dangerous gap between what developers think they are shipping (just the compiled or exported frontend files) versus what actually ends up on the server when entire local directories are blindly copied over. The real core lesson demonstrates how a tiny configuration slip-up transforms a static web application into an open-book audit of every architectural decision, internal comment, and historical misstep made by the engineering team.

## Exploitation / Walkthrough
### Step 1
Before breaking out heavy extraction tools, I performed a quick diagnostic check to verify if the .git directory was inadvertently exposed on the web server:

```bash
curl -i http://10.48.163.255:8080/.git/HEAD
```

This returned:
```
ref: refs/heads/main
```

The HEAD file acts as the ultimate canary. Because every standard git repository initializes with this file, receiving its literal contents instead of a 404 Not Found error definitively confirms that the entire hidden .git folder is publicly readable.

### Step 2
With public accessibility confirmed and directory listing disabled on the server, I turned to git-dumper to recursively harvest the raw git objects, reference pointers, and pack files, rebuilding a fully operational local repository from scratch:

```bash
pip install git-dumper --break-system-packages
git-dumper http://10.48.163.255:8080/.git/ ./bytelotus
```

Rather than merely auditing the active workspace or current HEAD snapshot, I performed a comprehensive historical review, traversing every commit across all branches and examining deep code diffs:

```bash
cd bytelotus
git log --all -p | less
```

- `--all` casts a wide net across every branch and reference, ensuring nothing missed on unmerged or experimental branches is overlooked.
- `-p` generates patch output, exposing the exact line-by-line modifications introduced in every historical commit.

Paging through the commit logs via less surfaced the target flag, which had been purged from the active branch in a later update but remained permanently etched into the object history.

Mission accomplished.

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000)

## Lessons Learned
Key takeaway: Deploying your raw project folder instead of a clean, compiled build is the digital equivalent of handing a burglar the blueprints, the spare keys, and a diary of every mistake you’ve ever made. When that .git folder hits production, it doesn’t just show the current state of your app; it exposes your entire commit history. Thinking you’re safe because you deleted an AWS key in a later commit is pure delusion. Git is a hoarder. Unless you aggressively nuke the timeline and rewrite history, that "deleted" secret is still sitting pretty in the object database, just waiting for anyone with extraction tools to come scoop it up.
