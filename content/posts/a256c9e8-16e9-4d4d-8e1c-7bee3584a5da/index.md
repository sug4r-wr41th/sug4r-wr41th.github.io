+++
title = 'The dark side of Vimeo: video meta-data contains PowerShell script and leads to host compromise'
date = 2023-10-10T22:03:53+02:00
draft = false
categories = ["Malware Analysis"]
tags = ["PowerShell"]
+++

# Introduction

During my daily blue-team operations I stumbled across a series of similar infection chains, all started with an infected thumb drive. Since the installed security solution detected with a generic signature the threat but didn't block it, I took charge of the alerts and decided to deep dive and analyze the sample.

# Analysis

The initial access vector is an infected USB stick, used to transfer files to a copy shop, containing the following files:

![](images/usb_drive_content.png)

- `E (9GB).lnk` it's a shortcut file
- `explorer.ps1` it's a PowerShell script, hidden
- `ㅤ` (`U+3164`) it's an empty folder, hidden

Opening `E (9GB).lnk`'s properties shows what seems obvious: the execution target is `explorer.ps1`. What seems odd it's that `E:` is not the mount point of the thumb drive on the victim machine; a good guess is that `E:` it's (or better, was) the mount point of the thumb drive on the infected machine, presumably a computer at the copy shop.

![](images/shortcut_properties.png)

Let's take a look at the PowerShell script:

![](images/0_stage_powershell_script.png)

which, using CyberChef or similar tools, can be easily deobfuscated by extracting, reversing, joining and converting the strings passed as argument to `FromBase64String` function:

![](images/0_stage_powershell_script_clean.png)

which leads to a first stage PowerShell script that, for the sake of simplicity, I've already deobfuscated:

![](images/1_stage_powershell_script_clean.png)

Reading trought the code reveals some interesting facts:

- the script makes a web request to Vimeo's streaming services, searching for a specific video meta-data; the response is extracted with a regular expression and decrypted using AES with the initialization vector and the symmetic key hard-coded into the script; the whole process undergoes an additional stage

![](images/vimeo_video.png)

![](images/vimeo_video_metadata.png)

- `44Wk` is the Base64 encoding of `ㅤ` (`U+3164`) which is exactly the name of the blank and hidden folder on the USB stick; in fact, the folder presence in pair with the command `Test-Path` is used to prevent the script from executing if the folder do not exists; a clever anti-sandbox technique

![](images/unicode_char.png)

[Hangul](https://en.wikipedia.org/wiki/Hangul) is the modern official writing system for the Korean language.

![](images/unicode_char_bin.png)

![](images/unicode_char_anti_sandbox.png)

To review the second stage, just print it instead of running it.

![](images/2_stage_powershell_script_clean.png)

Some findings:

- the folder trick is used again
- the `$uuid` variable is written to a file on disk
- the `$uuid` variable is appended to the drop url and a web request is made; the response is written to a file named `Runtime Broker.exe` and saved into the user temporary folder, then the script waits for five seconds
- if `%PROGRAMFILES%\WinSoft Update Service\pythonw.exe` exists the script terminates immediately, otherwise the script waits for one second then launches `Runtime Broker.exe`; this could be a kill switch implemented by the attacker during development in order to avoid auto-infection

At the time of analysis, I was unable to download the file but fortunately I managed to retrieve it from the security solution.

![](images/runtime_broker.png)

Analyzing the PE meta-data, specifically the description and icon, it's clear that the malware is trying to masquerade as Node.JS, an engine used to execute JavaScript code server-side.

![](images/pe_studio_analysis.png)

Detonating the file with [VirusTotal](https://www.virustotal.com/gui/file/a4f20b60a50345ddf3ac71b6e8c5ebcb9d069721b0b0edc822ed2e7569a0bb40), [Tria.ge](https://tria.ge/231011-hrrjwsae49) and [Hybrid Analysis](https://www.hybrid-analysis.com/sample/a4f20b60a50345ddf3ac71b6e8c5ebcb9d069721b0b0edc822ed2e7569a0bb40) did not lead to great discoveries but from threat intelligence sources the sample could be a variant of the open source infostealer TurkoRAT.

# Conclusion

As removable drives remain popular among users as an easy way to share files with colleagues and suppliers, attackers will continue to use them as infection vectors, using novel techniques to trick users into clicking on stuff.

# IOCs

| Description | Type | Value |
|-------------|------|-------|
|E (9GB).lnk|SHA256|c626e59bf85c006334ced82bda0b1a83fa720b0f455db1a040b47caf02a45b4c
|explorer.ps1|SHA256|8b1cd7700f0ca6cfd0bf1a291bf44074cd8fcbc6c0440a09d43422a5cc7fc3bb
|Vimeo video|URL|h++ps://vimeo.com/804838895
|Vimeo video|URL|h++ps://vimeo.com/api/v2/video/804838895.json
|Vimeo user|URL|h++ps://vimeo.com/user195838396
|Runtime Broker.exe|SHA256|a4f20b60a50345ddf3ac71b6e8c5ebcb9d069721b0b0edc822ed2e7569a0bb40
|.exe hosting infrastructure|domain|evinfeoptasw.dedyn[.]io