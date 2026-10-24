---
layout: "post"
title: "Skipping that 5$ coffee can save your job"
date: "2026-02-27 17:10:43 +0000"
description: "Turn an old Android phone into an AI agent host with Termux, SSH, PicoClaw, and Telegram."
categories: ["writing"]
tags: ["ai-agents", "android", "picoclaw"]
original_url: "https://x.com/pranay5255/status/2027431212864966725"
reading_time: 3
image: "/assets/posts/skipping-that-5-dollar-coffee-can-save-your-job/cover.jpg"
---

Bookmark this. I cannot stress this enough. BOOKMARK this post if you have an old android phone.

Every hiring manager I've talked to in the last 6 months asks the same thing: show me something you've built with AI. Not prompted. Built.
<strong>If you have an old Android phone, you already own a free AI agent.</strong> It has multicore processors, GBs of RAM, WiFi, and a battery backup that lasts days. Buying a VPS on average costs 5-10$, this is completely free.

## How to use this post

The sections are divided into the ones that you need to follow so its easier to just copy paste the code and let it run. Starting from getting a shell on the phone, to configuring the AI agent to use an API, to fixing the exact errors you'll hit on Android. Easy peasy setup.

---

## Why this works

<strong>PicoClaw exists now.</strong> An open-source AI agent written in Go that hit 12,000 GitHub stars in its first week. This is a simple clone of OpenClaw which works exactly the same.

<strong>Your old phone is absurdly overpowered for this.</strong>

My Moto G60 (8 years old) has a Snapdragon (8 cores), 6GB RAM, 128GB storage, and a 6000mAh battery. PicoClaw uses less than 20MB of RAM. A $5/month DigitalOcean droplet gives you 1 core and 512MB. Your drawer phone has 16x the memory and 8x the cores.

<strong>TERMUX :</strong> this is a simple app which gives you a terminal into your phone, this is exactly how we turn this into a <strong>"server".</strong>

---

## 1. Get shell on Phone

if you have a functional screen, then you can install termux from the app store directly, you'll need termux nonetheless for SSH connection. I used ADB because i don't want to type everything on phone.  Enable the following from phone settings:

<strong>Settings --> About Phone</strong> (at the bottom<strong>)--> Tap "Build Number" 7 times</strong>
This gives you access to developer settings.

<strong>Settings --> Developer Options --> Enable "USB Debugging"</strong>
This enables you to connect to the phone and install termux from the terminal.

1. Connect phone using USB cable to laptop

2. Select "File transfer" or "MTP", on the phone when asked.

3. open a terminal on laptop and paste the commands below

```bash

1. adb devices
2. adb shell
3. adb install termux.apk 

```

Until this point our objective is to setup and connect our phone to the laptop wirelessly.

---

## 2. Go Wireless

USB is for setup, <strong>SSH is forever.</strong> Also if your phone screen is unusable and you don't have an alternative. Make sure that your phone and the laptop are on the same Wifi network.

<strong>Inside TERMUX on phone -</strong>

```bash
pkg update -y && pkg upgrade -y
pkg install -y openssh

# Set SSH password
passwd

#Find user id
whoami
# Output will be something like: u0_a234

# Start SSH server
sshd

# Find phone's IP
ifconfig wlan0 | grep "inet "
```

<strong>From Your Computer -</strong>

```bash
# Replace u0_a234 and 192.168.1.42 with your actual whoami and IP address
ssh -p 8022 u0_a234@192.168.1.42


# Termux SSH runs on PORT 8022 by default

# Add to ~/.ssh/config on your computer
Host phone
    HostName 192.168.1.42
    Port 8022
    User u0_a234
```

---

## 3. Get secret API tokens

In this step you'll have acquire three different secret tokens, all of them are going to be free to acquire. 
- First one is the openrouter.ai/keys API key, this is for the  LLM API which the agent uses for intelligence. Thank @OpenRouter for free tokens
- Telegram BOT token - Open your telegram and search "BotFather" , message "/newbot" in this chat. Copy the token. 
- Get you Telegarm User ID by messaging "@userinfobot"

**<strong>RECORD THESE TOKENS in a .txt **</strong> 
We will need them later.

---

## 4. Install PicoClaw

If the SSH connection above {step 2} succeeds then you have terminal access into your phone and now is the <strong>time to</strong> <strong>EAT.</strong>  Next few commands are going to setup picoclaw.

```bash
pkg install -y proot wget tar

cd ~
wget https://github.com/sipeed/picoclaw/releases/download/v0.1.2/picoclaw_Linux_arm64.tar.gz
tar -xzf picoclaw_Linux_arm64.tar.gz
chmod +x picoclaw
rm picoclaw_Linux_arm64.tar.gz

# Initialize
proot ./picoclaw onboard
# Output: 🦞 picoclaw is ready!
```

Use the .txt with the secret tokens in the next few steps

---

## 5. Setup Configs

<strong>5.1 Set Openrouter key and Model</strong>

```bash
sed -i 's/"provider": ""/"provider": "openrouter"/' ~/.picoclaw/config.json
sed -i 's/"model": "glm-4.7"/"model": "minimax\/minimax-m2.5"/' ~/.picoclaw/config.json

# Replace sk-or-v1-YOUR-KEY with your actual key
sed -i '/"openrouter"/{n;s/"api_key": ""/"api_key": "sk-or-v1-YOUR-KEY"/}' ~/.picoclaw/config.json
```

<strong>5.2 Set Telegram bot key and enable it</strong>

```bash
sed -i '/"telegram"/{n;s/"enabled": false/"enabled": true/}' ~/.picoclaw/config.json
sed -i '/"telegram"/{n;n;s/"token": ""/"token": "YOUR_BOT_TOKEN"/}' ~/.picoclaw/config.json
sed -i '/"telegram"/,/"allow_from"/{s/"allow_from": \[\]/"allow_from": ["YOUR_USER_ID"]/}' ~/.picoclaw/config.json
```

---

## 6. Troubleshooting | Fix DNS and TLS

Sometimes, you'll get errors which look like this

```bash
dial tcp: lookup openrouter.ai on [::1]:53: connection refused

#OR

tls: failed to verify certificate: x509: certificate signed by unknown authority
```

Just paste these snippets to fix them

```bash
#DNS fix
mkdir -p ~/etc
echo "nameserver 8.8.8.8" > ~/etc/resolv.conf
echo "nameserver 1.1.1.1" >> ~/etc/resolv.conf

#TLS fix
proot \
  -b ~/etc/resolv.conf:/etc/resolv.conf \
  -b /etc/security/cacerts:/etc/ssl/certs \
  ./picoclaw agent -m "Hello!"
```

---

## 7. Launch Bot

This is the final step.

```bash

proot \
  -b ~/etc/resolv.conf:/etc/resolv.conf \
  -b /etc/security/cacerts:/etc/ssl/certs \
  ./picoclaw agent -m "Hello!"
```

Expected output --

```bash
🦞 Hello! 👋
I'm PicoClaw, your lightweight AI assistant. How can I help you today?
```

If this succeeds then you need to Start the Telegram Bot which runs forevaa

```bash
proot \
  -b ~/etc/resolv.conf:/etc/resolv.conf \
  -b /etc/security/cacerts:/etc/ssl/certs \
  ./picoclaw gateway
```

---

There are <strong>billions</strong> of old phones in drawers worldwide. Each one has more compute than most of the servers that ran the early internet. The openclaw ecosystem, including Picoclaw is a bet towards open, sovereign and hardware owned AI.

I believe in that bet.

I'm Pranay. I build open-source AI x Crypto and i think this intersection is where the hardest, most meaningful problems live. 
This Agent? It's a small version of that future. Your model choice. Your data. Your hardware. No monthly invoice from AWS. No permission needed from anyone.

The tools exist now, the hardware is in your drawer.

Are you ready to experiment with AI anon?

🦞🦞🦞
