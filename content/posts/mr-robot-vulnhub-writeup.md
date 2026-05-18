+++
date = '2026-05-18T15:48:46+08:00'
draft = true
title = 'Mr Robot Vulnhub Writeup'
+++

---
title: "Mr. Robot VulnHub Walkthrough"
date: 2026-05-18T15:30:00+08:00
draft: false
tags: ["CTF", "VulnHub", "PrivEsc"]
categories: ["Walkthrough"]
---

## Executive Summary
Mesin Mr. Robot dari VulnHub adalah platform CTF yang menguji kemahiran pengetesan penembusan (penetration testing). Mesin ini mempunyai 3 key/flag yang perlu dicari menerusi enumeration, exploitation, dan privilege escalation.

## 1. Reconnaissance (Nmap)
Command yang digunakan untuk fasa awal:
```bash
nmap -sV -sC -A -T4 192.168.1.X

### Test Guna Local Server
Sebelum hantar ke internet, kau boleh tengok rupa website kau kat komputer sendiri:
```bash
hugo server