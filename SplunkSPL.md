
# Splunk SPL Investigation Report

## Introduction

Boss of the SOC (BOTS) v1 gives analysts two realistic incidents to investigate in Splunk. The first is the defacement of *imreallynotbatman.com* by the Po1s0n1vy group. The second is a Cerber ransomware outbreak on the workstation *we8105desk*. This walkthrough follows both investigations question by question. Each section gives the reasoning, the SPL, a line-by-line explanation of the search, and the answer.

For every search, set the time range to **All time**, because the dataset dates from 2016.

<img width="633" height="668" alt="image" src="https://github.com/user-attachments/assets/a192ba22-07ca-49f3-9ab6-1eff42d4cee7" />


# Part 1: Web Site Defacement

## 101.Question: What is the likely IPv4 address of someone from the Po1s0n1vy group scanning imreallynotbatman.com for web application vulnerabilities?

The first step in a web-attack investigation is to see who is talking to the target. Filtering on POST requests to the site and counting them per source IP shows which hosts are sending unusual volumes.

```spl
index=botsv1 imreallynotbatman.com http_method=POST
| stats count by src_ip
```

**SPL Explanation**

* `index=botsv1`: searches the Boss of the SOC dataset.
* `imreallynotbatman.com`: filters events related to the target website.
* `http_method=POST`: limits results to HTTP POST requests.
* `|`: passes the results to the next command.
* `stats count by src_ip`: counts the requests from each source IP.

The search returned 32,899 events across three source IPs. `40.80.148.42` generated 26,465 requests, far more than `23.22.63.114` (1,236) or the internal web server `192.168.250.70` (5,196). That volume is the signature of an automated scanner.

**Answer:** `40.80.148.42`

<img width="900" height="401" alt="image" src="https://github.com/user-attachments/assets/5009cbe7-77b0-439b-bbaa-f52cc8c247c3" />


## 102.Question: What company created the web vulnerability scanner used by Po1s0n1vy? Type the company name.

Scanners rarely hide. Their requests usually carry headers that name the tool. Narrowing the previous search to the scanner's IP and opening the `src_headers` field shows those headers.

```spl
index=botsv1 imreallynotbatman.com http_method=POST src_ip=40.80.148.42
```

**SPL Explanation**

* `src_ip=40.80.148.42`: isolates the scanner's traffic (26,465 events).
* `src_headers` (field sidebar → Top values): shows the raw request headers, where the tool identifies itself.

The top values contain Acunetix's product tag, *Acunetix Web Vulnerability Scanner – Free Edition*, and its scanning-agreement notice. The requests target Joomla search URLs.

**Answer:** `Acunetix`

<img width="900" height="592" alt="image" src="https://github.com/user-attachments/assets/89191b03-b7a7-4e71-9cec-487786bec103" />


<img width="900" height="594" alt="image" src="https://github.com/user-attachments/assets/c57f8f33-50ed-42f5-857c-3889c8d0030f" />


## 103.Question: What content management system is imreallynotbatman.com likely using?

The same header values also reveal the CMS. The scanner's requests target paths such as `/joomla/index.php/component/search/`. The `/joomla/` folder and the `/component/` path structure are characteristic of Joomla. No new search is needed.

**Answer:** `joomla`

<img width="900" height="594" alt="image" src="https://github.com/user-attachments/assets/bd329c85-83cb-4080-bb97-9a1ea48650e0" />


## 104.Question: What is the name of the file that defaced the website?

A web server normally answers requests and doesn't make them. Searching for HTTP traffic where the server `192.168.250.70` is the **source** surfaces anything it fetched from the outside.

```spl
index=botsv1 sourcetype=stream:http src_ip=192.168.250.70
```

**SPL Explanation**

* `sourcetype=stream:http`: captured HTTP traffic.
* `src_ip=192.168.250.70`: requests initiated by the web server.
* `src_headers` (field sidebar): shows what it requested.

Only 8 events match. Most are update checks to *update.joomla.org*. The outlier is a GET for `/poisonivy-is-coming-for-you-batman.jpeg`, the file used to deface the site.

**Answer:** `poisonivy-is-coming-for-you-batman.jpeg`

<img width="900" height="541" alt="image" src="https://github.com/user-attachments/assets/e0097d58-3b08-4715-929b-7693b7f727c2" />


<img width="900" height="549" alt="image" src="https://github.com/user-attachments/assets/fdba68a4-e95e-4380-8415-701e48d90b13" />


## 105.Question: This attack used dynamic DNS to resolve to the malicious IP. What fully qualified domain name (FQDN) is associated with this attack?

The defacement request also exposes where the file came from. Its Host header shows the dynamic DNS name `prankglassinebracket.jumpingcrab.com:1337`. Dynamic DNS lets an attacker point a memorable name at any IP address and change it on demand. No new search is needed.

**Answer:** `prankglassinebracket.jumpingcrab.com`

<img width="900" height="549" alt="image" src="https://github.com/user-attachments/assets/3a38d1a6-dbb7-4ed3-874a-4a5b64beb45e" />


## 106.Question: What IPv4 address has Po1s0n1vy tied to domains that are pre-staged to attack Wayne Enterprises?

To find the IP behind that domain, check the `dest_ip` field of the web server's outbound traffic and then filter on the candidate.

```spl
index=botsv1 sourcetype=stream:http src_ip=192.168.250.70 dest_ip="23.22.63.114"
```

**SPL Explanation**

* `src_ip` and `dest_ip` together: traffic from the web server to a single external address.
* `src_headers`: confirms what was requested.

The `dest_ip` field shows two external destinations, `108.161.187.134` (6 events) and `23.22.63.114` (2 events). Filtering on `23.22.63.114` returns 2 events with a single header value, the defacement GET. This IP is part of the attacker's infrastructure and links the web server's request to the **pre-staged domain** used to deliver the defacement file.

**Answer:** `23.22.63.114`

<img width="900" height="524" alt="image" src="https://github.com/user-attachments/assets/aa04aec5-3ec8-4ff4-8fb8-a69669911499" />


<img width="900" height="522" alt="image" src="https://github.com/user-attachments/assets/ed7de47a-43b2-43fd-86a8-25f5e5ec3425" />

## 108.Question:What IPv4 address is likely attempting a brute force password attack against imreallynotbatman.com?

Brute forcing appears as a large number of submissions to one login page. Counting POSTs to the web server by IP and URI makes it obvious.

```spl
index=botsv1 sourcetype=stream:http http_method=POST dest_ip=192.168.250.70
| stats count by src_ip, uri
```

**SPL Explanation**

* `http_method=POST` and `dest_ip=192.168.250.70`: form submissions to the web server.
* `stats count by src_ip, uri`: requests per IP and page.

The search returned 15,560 events in 73 rows. `23.22.63.114` made 412 requests to `/joomla/administrator/index.php`, the Joomla login page. By contrast, `40.80.148.42` made single requests to many different URIs, including test upload paths containing `acunetix_test`, which is scanner behavior rather than password guessing.

**Answer:** `23.22.63.114`

<img width="900" height="397" alt="image" src="https://github.com/user-attachments/assets/6298bbb8-0b8b-4b26-94f2-e1e42460f060" />


## 109.Question: What is the name of the executable uploaded by Po1s0n1vy?

File uploads are sent as multipart POST requests, and Splunk stores the uploaded names in the multivalue field `part_filename{}`.

```spl
index=botsv1 sourcetype=stream:http dest_ip=192.168.250.70 http_method=POST
```

**SPL Explanation**

* `part_filename{}` (field sidebar → Top values): the names of uploaded files. The braces mark it as multivalue.

The field has two values, each seen once: `3791.exe` and `agent.php`. The executable is `3791.exe`.

**Answer:** `3791.exe`

<img width="900" height="592" alt="image" src="https://github.com/user-attachments/assets/931cac10-91e4-446b-b8d5-6667ffd4206e" />

<img width="900" height="519" alt="image" src="https://github.com/user-attachments/assets/6a152e14-f6f9-4efb-ac46-7175918a7dba" />


## 110.Question: What is the MD5 hash of the executable uploaded?

Uploading a file proves nothing until it runs. Searching for the file name across the dataset shows which log sources recorded it.

```spl
index=botsv1 3791.exe
| stats count by sourcetype
```

The search returns 76 events across five sources: Windows Security, Sysmon, FortiGate (`fgt_utm`), HTTP stream, and Suricata. Sysmon is the source that records file hashes, so the next search narrows to it:

```spl
index=botsv1 3791.exe sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" CommandLine="3791.exe"
```

**SPL Explanation**

* `3791.exe`: free-text filter for the file name.
* `sourcetype="...Sysmon/Operational"`: endpoint process telemetry. This gives 69 events from host `we1149srv`.
* `CommandLine="3791.exe"`: the exact process-creation event.


**Answer:** `AAE3F5A29935E6ABCC2C2754D12A9AF0`

<img width="900" height="692" alt="image" src="https://github.com/user-attachments/assets/d642cc55-5c70-4edf-b975-480fe1cd7857" />

<img width="900" height="549" alt="image" src="https://github.com/user-attachments/assets/09910c78-4ac0-4c05-87fb-b73345dd71d1" />

<img width="900" height="584" alt="image" src="https://github.com/user-attachments/assets/0944ef79-3996-4dec-9fd4-4c38d756efbe" />

<img width="900" height="402" alt="image" src="https://github.com/user-attachments/assets/78367be8-8323-4fae-9b06-e34c4b32eced" />


## 111.Question: GCPD reported that common TTPs (Tactics, Techniques, Procedures) for the Po1s0n1vy APT group, if initial compromise fails, is to send a spear phishing email with custom malware attached to their intended target. This malware is usually connected to Po1s0n1vys initial attack infrastructure. Using research techniques, provide the SHA256 hash of this malware.

Some questions can't be answered from logs alone. The Sysmon results show that the `dest_ip` for this activity is a single address, `23.22.63.114`. Pivoting on that IP in VirusTotal shows it belongs to Amazon AS14618 and lists **5 communicating files**: `software.exe`, `MirandaTateScreensaver.scr.exe`, `MSRSAAPP`, `etfdbaf5x.exe`, and `ab.exe`. The screensaver name fits the Miranda Tate theme of the scenario. Its report shows 48 of 70 vendors flagging it as malicious, with labels such as `trojan.redsip/sanwaicrypt`.

**Answer:** `9709473ab351387aab9e816eff3910b9f28a7a70202e250ed46dba8f820f34a8`

<img width="900" height="547" alt="image" src="https://github.com/user-attachments/assets/579911a3-05cf-4324-9d1e-84eb396593e3" />

<img width="900" height="403" alt="image" src="https://github.com/user-attachments/assets/07e8554c-e5ea-420b-9485-ff0a8b89ecb7" />

<img width="816" height="441" alt="image" src="https://github.com/user-attachments/assets/ddcc348d-897f-4dcf-be8a-7ec11076d0a5" />

<img width="900" height="455" alt="image" src="https://github.com/user-attachments/assets/1e378c55-dd1c-4e45-a60b-e63a7bf6fa2a" />


## 112.Question: What special hex code is associated with the customized malware discussed in question 111?

The hint says the value isn't in Splunk. The place to look is the **Community** tab of the same VirusTotal report. It shows the file in six community graphs (one named *botsv1-poison-ivy*), and the comments include a string of hex bytes. Decoded, it reads: *"Steve Brant's Beard is a powerful thing. Find this message and ask him to buy you a beer!!!"* The trailing "Adham was here :)" is a separate note and not part of the hex.

**Answer:**

```text
53 74 65 76 65 20 42 72 61 6e 74 27 73 20 42 65 61 72 64 20 69 73 20 61 20 70 6f 77 65 72 66 75 6c 20 74 68 69 6e 67 2e 20 46 69 6e 64 20 74 68 69 73 20 6d 65 73 73 61 67 65 20 61 6e 64 20 61 73 6b 20 68 69 6d 20 74 6f 20 62 75 79 20 79 6f 75 20 61 20 62 65 65 72 21 21 21
```

<img width="900" height="618" alt="image" src="https://github.com/user-attachments/assets/90886533-1da9-4e5a-9eac-79bca4780dce" />

<img width="900" height="277" alt="image" src="https://github.com/user-attachments/assets/8672bc02-aa1b-4f25-96dd-23f9c7cb4dc2" />



## 114.Question: What was the first brute force password used?

To find the earliest attempt, extract the password from the raw form data and sort by time.

```spl
index=botsv1 sourcetype=stream:http src_ip=23.22.63.114 http_method=POST form_data=*passwd*
| rex field=form_data "passwd=(?<pass>[^&]+)"
| sort 0 _time
| table _time pass
```

**SPL Explanation**

* `form_data=*passwd*`: login POSTs only (412 events).
* `rex field=form_data "passwd=(?<pass>[^&]+)"`: captures the text after `passwd=` up to the next `&` into a new field called `pass`.
* `sort 0 _time`: oldest first, with no row limit.
* `table _time pass`: time and password columns.

The first row, at 21:45:21.226 on `2016-08-10`, is `12345678`. It is followed by `letmein`, `qwerty`, `1234`, `123456`, and `football`.

**Answer:** `12345678`

<img width="900" height="436" alt="image" src="https://github.com/user-attachments/assets/5fd545c7-1b51-4a51-bad9-a32b1aa38e1b" />

or

<img width="900" height="451" alt="image" src="https://github.com/user-attachments/assets/2ae01299-6ed5-4551-8625-8df7f5505996" />


## 115.Question: One of the passwords in the brute force attack is James Brodsky's favorite Coldplay song. We are looking for a six character word on this one. Which is it?

The clue narrows the field to six-letter words. The search extracts six-letter alphabetic passwords and compares them with a short list of Coldplay-related candidates.

```spl
index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" src_ip="23.22.63.114" http_method=POST
| rex field=form_data "(?i)passwd=(?<string>[a-zA-Z]{6})(?=&|$)"
| search string IN (Aliens, Broken, Church, Clocks, Murder, Oceans, Shiver, Sparks, Wizkid, Yellow)
| table src_ip string
```

**SPL Explanation**

* `(?i)`: case-insensitive match.
* `[a-zA-Z]{6}`: exactly six letters.
* `(?=&|$)`: the password must end at the next `&` or at the end of the string.
* `search string IN (...)`: keeps only matches from the candidate list.

Exactly one event matches, from `23.22.63.114`.

**Answer:** `yellow`

<img width="900" height="384" alt="image" src="https://github.com/user-attachments/assets/6a3bc3ab-32da-4656-a6e6-8ffc6ab957c9" />


## 116.Question: What was the correct password for admin access to the content management system running "imreallynotbatman.com"?

The attack had two stages: the brute forcer found the password, and the intruder then logged in with it. Counting every submitted password across all IPs makes the repeated one stand out.

```spl
index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" http_method=POST
| rex field=form_data "passwd=(?<string>\w+)"
| stats count by string
| sort - count
| table string, count
```

**SPL Explanation**

* No `src_ip` filter, so both the brute forcer and the intruder are included.
* `stats count by string` and `sort - count`: passwords used more than once rise to the top.

The search returns 412 distinct passwords. `batman` appears twice, while every other password appears once.

**Answer:** `batman`

<img width="900" height="372" alt="image" src="https://github.com/user-attachments/assets/ab8ba5e6-64d3-4bb0-b5f3-18c0ab575f7e" />


## 117.Question: What was the average password length used in the password brute forcing attempt?

```spl
index=botsv1 src_ip=23.22.63.114 form_data=*passwd*
| rex field=form_data "passwd=(?<pw>[^&]+)"
| eval strlen=len(pw)
| stats avg(strlen)
```

**SPL Explanation**

* `eval strlen=len(pw)`: character count per password.
* `stats avg(strlen)`: mean across all 412 attempts.

The result is 6.1747…, which rounds to 6.

**Answer:** `6`

<img width="897" height="682" alt="image" src="https://github.com/user-attachments/assets/be5dc774-84de-42ea-b8e5-8336f288fe26" />


## 118.Question: How many seconds elapsed between the time the brute force password scan identified the correct password and the compromised login?

Both events that used `batman` can be grouped into one transaction, and Splunk reports the time span as `duration`.

```spl
index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" http_method=POST form_data=*passwd*batman*
| rex field=form_data "passwd=(?<string>\w+)"
| transaction string
| table duration
```

**SPL Explanation**

* `form_data=*passwd*batman*`: only attempts using `batman`.
* `transaction string`: combines events sharing the same password into one group.
* `table duration`: the time from the first to the last event in the group.

The duration is 92.169084 seconds.

**Answer:** `92.17`

<img width="900" height="513" alt="image" src="https://github.com/user-attachments/assets/7d861d32-9c72-43ae-af85-f12a84244cd6" />


## 119.Question: How many unique passwords were attempted in the brute force attempt?

```spl
index=botsv1 src_ip=23.22.63.114 form_data=*passwd*
| rex field=form_data "passwd=(?<pw>[^&]+)"
| stats dc(pw)
```

**SPL Explanation:** `dc(pw)` is a distinct count, so repeated passwords count once. The result equals the number of events, which means every attempt used a different password.

**Answer:** `412`

<img width="900" height="516" alt="image" src="https://github.com/user-attachments/assets/da38c736-b5e1-449d-b6cd-e7e5aab95c5c" />





# Part 2: Ransomware

## 200.Question: What was the most likely IPv4 address of we8105desk on 24AUG2016?

To find the machine's address on a given day, count its events by source IP within that date window.

```spl
index=botsv1 host="we8105desk" earliest="08/24/2016:00:00:00" latest="08/25/2016:00:00:00"
| stats count by src_ip
| sort - count
```

**SPL Explanation**

* `host="we8105desk"`: the workstation.
* `earliest=` / `latest=`: limits the search to 24 August 2016.
* `stats count by src_ip` and `sort - count`: the most common address comes first.

The search covers 179,081 events in 7 rows. `192.168.250.100` dominates with 52,270 events, well ahead of `192.168.2.50` (1,217). The rest are broadcast, loopback, and empty addresses.

**Answer:** `192.168.250.100`

<img width="900" height="404" alt="image" src="https://github.com/user-attachments/assets/bf76c4a0-d6f1-4553-badb-bed99485927b" />


## 201.Question: Amongst the Suricata signatures that detected the Cerber malware, which one alerted the fewest number of times? Submit ONLY the signature ID value as the answer.

Suricata raises alerts from signatures. Counting alerts per Cerber rule and sorting ascending answers the question.

```spl
index=botsv1 sourcetype=suricata cerber
| stats count by alert.signature_id, alert.signature
| sort count
```

**SPL Explanation:** `sort count` without a minus sign is ascending, so the rarest signature is first.

| Signature ID | Name | Alerts |
|---|---|---:|
| 2816763 | ETPRO TROJAN Ransomware/Cerber Checkin 2 | 1 |
| 2816764 | ETPRO TROJAN Ransomware/Cerber Checkin Error ICMP Response | 2 |
| 2820156 | ETPRO TROJAN Ransomware/Cerber Onion Domain Lookup | 2 |

**Answer:** `2816763`

<img width="900" height="360" alt="image" src="https://github.com/user-attachments/assets/be349311-d0b4-487e-9d4e-50ebc8b2c170" />


## 202.Question: What fully qualified domain name (FQDN) does the Cerber ransomware attempt to direct the user to at the end of its encryption phase?

The workstation's DNS lookups list 82 domains, mostly Microsoft and other normal hosts. One stands out, and a wildcard search confirms it.

```spl
index=botsv1 sourcetype=stream:dns src_ip=192.168.250.100 query{}=*cerber*
| table _time, query{}
```

**SPL Explanation:** `query{}=*cerber*` is a wildcard on the DNS query name, and `table` shows its time and value. A single event at 17:15:12 on 24 August points to the domain below.

**Answer:** `cerberhhyed5frqa.xmfir0.win`

<img width="900" height="692" alt="image" src="https://github.com/user-attachments/assets/2a284d45-3175-48fc-8c90-165d0ff7ffa8" />


## 203.Question: What was the first suspicious domain visited by we8105desk on 24AUG2016?

The noise in a workstation's DNS log can be removed with `NOT` filters, leaving a chronological list that an analyst can read.

```spl
index=botsv1 sourcetype=stream:dns src_ip=192.168.250.100 record_type=A NOT query{}=*microsoft* NOT query{}=*windows* NOT query{}=*.local NOT query{}=*acronis* NOT query{}=*bing*
| table _time, query{}
| sort 0 _time
```

**SPL Explanation**

* `NOT query{}=...`: removes known-good domains (89 events remain).
* `sort 0 _time`: chronological order.

Early rows from 10 August show `isatap`. On 24 August, after the normal `dns.msftncsi.com` check at 16:34:39, the first unfamiliar domain is at 16:48:12. It is followed by `ipinfo.io` at 16:49:24 and the Cerber domain at 17:15:12.

**Answer:** `solidaritedeproximite.org`

<img width="900" height="441" alt="image" src="https://github.com/user-attachments/assets/f0a95842-f308-4649-9ac4-b6a1f54716ee" />

<img width="900" height="611" alt="image" src="https://github.com/user-attachments/assets/ea5548f8-c1a3-4c4e-baf3-e8dac7662873" />


## 204.Question: During the initial Cerber infection a VB script is run. The entire script from this execution, pre-pended by the name of the launching .exe, can be found in a field in Splunk. What is the length of the value of this field?

Sysmon process-creation events record the full command line, and `len()` gives its length.

```spl
index=botsv1 host=we8105desk sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational EventCode=1 *.vbs
| eval strlen=len(CommandLine)
| table _time, strlen, CommandLine
```

**SPL Explanation**

* `EventCode=1`: process creation.
* `*.vbs`: events involving a VBScript file.
* `eval strlen=len(CommandLine)`: counts characters.

Ten events match. The obfuscated `cmd.exe /V /C set "GSI=%APPDATA%\%RANDOM%.vbs" …` command at 16:43:21 is 4,490 characters long. The other rows are short: 100, 102, and 93.

**Answer:** `4490`

<img width="900" height="400" alt="image" src="https://github.com/user-attachments/assets/02ccbe9d-87df-4d0e-89b8-3f8184eb1c32" />


## 205.Question: What is the name of the USB key inserted by Bob Smith?

Windows stores a USB device's friendly name in the registry, and Splunk collects that data.

```spl
index=botsv1 host=we8105desk sourcetype=winregistry friendlyname
| table _time, registry_key_name, registry_value_data
```

**SPL Explanation:** `friendlyname` matches the registry value holding the device's display name. `table` shows the key and its data.

Two events at 16:42:17 show USBSTOR entries for a generic flash disk. Both have the same value.

**Answer:** `MIRANDA_PRI`

<img width="900" height="243" alt="image" src="https://github.com/user-attachments/assets/af8a5d49-1622-4085-aa13-ae6f881bed47" />


## 206.Question: Bob Smith's workstation (we8105desk) was connected to a file server during the ransomware outbreak. What is the IPv4 address of the file server?

SMB is the protocol for Windows file sharing, so counting SMB traffic by destination identifies the file server.

```spl
index=botsv1 src_ip=192.168.250.100 sourcetype=stream:smb
| stats count by dest_ip
```

**SPL Explanation:** `stats count by dest_ip` counts SMB events per destination.

Of 39,304 events, `192.168.250.20` accounts for 39,204. The other two destinations (`192.168.2.50` and the broadcast address `192.168.250.255`) have 24 and 76.

**Answer:** `192.168.250.20`

<img width="900" height="283" alt="image" src="https://github.com/user-attachments/assets/8aa153af-f85f-4f1c-9f0a-30f5ec3b3a68" />


## 207.Question: How many distinct PDFs did the ransomware encrypt on the remote file server?

Searching the file server's own Security log gives a precise count of the PDFs touched by the workstation.

```spl
index=botsv1 sourcetype="WinEventLog:Security" host=we9041srv *.pdf Source_Address="192.168.250.100"
| stats dc(Relative_Target_Name) as totalcount
```

**SPL Explanation**

* `host=we9041srv`: the file server.
* `*.pdf`: events mentioning PDFs. A broader search returned 526 events with share path `C:\fileshare` and names like `996\996339.pdf`.
* `Source_Address="192.168.250.100"`: accesses from the infected PC.
* `dc(Relative_Target_Name)`: unique file names.

**Answer:** `257`

<img width="900" height="469" alt="image" src="https://github.com/user-attachments/assets/b9bf56e0-5a0d-4b94-bdf0-151d64073edf" />

<img width="900" height="383" alt="image" src="https://github.com/user-attachments/assets/3e2fecde-0733-41c6-8b2b-f2a4fc253139" />


## 208.Question: The VBscript found in question 204 launches 121214.tmp. What is the ParentProcessId of this initial launch?

Sysmon records both the new process and its parent, so the process chain can be read directly.

```spl
index=botsv1 host=we8105desk sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational EventCode=1 121214.tmp
| table _time, ParentProcessId, ProcessId, ParentCommandLine, CommandLine
| sort 0 _time
```

**SPL Explanation:** `ParentProcessId` is the launcher's ID and `ProcessId` is the new process's ID.

Seven events match. At 16:48:21, `WScript.exe` (process ID **3968**) running `20429.vbs` launches `cmd.exe /C START "" ...121214.tmp` (process 1476). `cmd.exe` then starts `121214.tmp` itself (process 2948, parent 1476). The initial launch from the VBScript therefore has ParentProcessId 3968.

**Answer:** `3968`

<img width="900" height="406" alt="image" src="https://github.com/user-attachments/assets/7caaa278-2ee7-41c6-8b7b-8313d126786b" />


## 209.Question: The Cerber ransomware encrypts files located in Bob Smith's Windows profile. How many .txt files does it encrypt?

```spl
index=botsv1 host=we8105desk sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational TargetFilename="C:\\Users\\bob.smith.WAYNECORPINC\\*.txt"
| stats dc(TargetFilename)
```

**SPL Explanation**

* `TargetFilename="C:\\Users\\bob.smith.WAYNECORPINC\\*.txt"`: `.txt` files in Bob's profile, with backslashes doubled as escapes.
* `stats dc(TargetFilename)`: counts each file once.

Splunk warns that a wildcard in the middle of a string can give inconsistent results, but the search returned 406 events and a distinct count of 406.

**Answer:** `406`

<img width="900" height="331" alt="image" src="https://github.com/user-attachments/assets/898b9f7d-9f8d-4bc6-bdaf-63cd9f4f750f" />


## 210.Question: The malware downloads a file that contains the Cerber ransomware cryptor code. What is the name of that file?

The Cerber payload arrived from the suspicious domain identified in question 203.

```spl
index=botsv1 sourcetype=suricata dest_ip=192.168.250.100 http.hostname="solidaritedeproximite.org"
| stats count by http.url
```

**SPL Explanation:** `http.hostname` filters to the domain, and `stats count by http.url` lists the files served to the workstation.

One event matches, and the URL is an image file.

**Answer:** `mhtr.jpg`

<img width="900" height="371" alt="image" src="https://github.com/user-attachments/assets/4f92ede0-1c15-403a-a8ff-b0ac4e1462aa" />


## 211.Question: Now that you know the name of the ransomware's encryptor file, what obfuscation technique does it likely use?

A payload delivered as a `.jpg` that actually carries the encryptor hides its content inside an image. That technique is **steganography**.

**Answer:** `steganography`

---

