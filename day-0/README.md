# Day 0

- **Challenge name:** [The Brochure](https://tryhackme.com/room/hh-thebrochure-081f3e36)
- **Category:** OSINT
- **Difficulty:** Easy
- **Date completed:** 25th July 2026

## Summary

This challenge is about downloading and reviewing a given task file - in this case, a Byte Lotus Hotel brochure saved as a `.png` file. The clues to finding the flag are hidden somewhere in the brochure.

## Walkthrough

### Step 1

Right after opening the file, I went straight to zooming in to look for watermarks and dug into the file properties to see if anything weird was hiding in the metadata. Spoiler alert: I came up completely empty-handed.

Then I remembered that this challenge is about **OSINT**, and specifically about using Social Media & Username Intelligence (**SOCMINT**) techniques, as I would later find out. Looking at the brochure again, I noticed a clue that read: *"Find us on Instagram... or not."*
<img width="2816" height="1534" alt="image" src="https://github.com/user-attachments/assets/0779c4f8-74ce-46ba-bdd0-2011d30fdb4a" />

<img width="2843" height="1543" alt="image" src="https://github.com/user-attachments/assets/eedc6fea-2b7d-4d2d-b68d-6d1fda907062" />

I opened Instagram and searched for **Byte Lotus Resort**, and found an account called **thebytelotusresort**. The account had 2 pictures uploaded as of 25th of July, but what caught my eye was that it followed exactly 1 other account, called *veratheconcierge*.


I opened the account **veratheconcierge**, visible in the image above. This account had 3 posts, each containing a string of upper- and lower-case letters, numbers, and symbols:

- `VEhNe1YzckBzX2FD`
- `QzB1bnRfaDRzX2Iz`
- `M25fZjB1bmQhfQ==`
### Step 3

These three strings are Base64-encoded. Decoding each part and joining them together gives the flag:
- `VEhNe1YzckBzX2FD` = ![redacted](https://img.shields.io/badge/-REDACTED-000000)
- `QzB1bnRfaDRzX2Iz` = ![redacted](https://img.shields.io/badge/-REDACTED-000000)
- `M25fZjB1bmQhfQ==` = ![redacted](https://img.shields.io/badge/-REDACTED-000000)

> **Note:** For decoding I used a bash terminal command:
>
> 'echo -n "code-to-be-decoded" | base64 -d'   


## Flag 
![redacted](https://img.shields.io/badge/-REDACTED-000000) 


## Lessons Learned

Main Takeaway: Never underestimate the "insignificant" details. A standard marketing brochure can easily leak breadcrumbs—like stray hints or connected social profiles—straight to sensitive data. Always audit metadata, hidden text, and social footprints before writing off a target.
