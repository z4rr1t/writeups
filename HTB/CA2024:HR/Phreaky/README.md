[](ctf=cyber-apocalypse-2024-hacker-royale)
[](type=forensics)
[](tags=game)
[](techniques=pcap-analysis)

# Phreaky.hard (forensics-300)
This is a treacky challenge, it requires a bit of python programming skills to solve it quickly, or without them to take forever

## Resources
challenge file is a capture file: phreaky.pcap  

![](https://github.com/z4rr1t/writeups/blob/main/HTB/CA2024%3AHR/Phreaky/img/pcap.png?raw=true)  

## Analysis
First of all i went to protocol hierarchy to see which communication I am dealing with, and which protocol. i found out that SMTP serves as application layer for TCP/IP  

![](https://github.com/z4rr1t/writeups/blob/main/HTB/CA2024%3AHR/Phreaky/img/protocol-hierarchy.png?raw=true)  

Later on, I filtered in SMTP protocol packets to focus on SMTP traffic. I identified IMF headers in the network traffic, indicating potential abnormalities or security concerns. I filtered in SMTP traffic with IMF headers,  I narrowed down the packets to a total of 15. These packets contain Base64-encoded strings, which appear to be fragments of ZIP files password protected.  

![](https://github.com/z4rr1t/writeups/blob/main/HTB/CA2024%3AHR/Phreaky/img/imf1.png?raw=true)  

by extracting the first packet, the packets contains pdf file decomposed to 15 fragments  

## Solve
I am a lazy person and i wont go through every base64 encoded zip file manually and unzip it, so i crafetd this python script using pyshark:  
```python
import pyshark
from binascii import unhexlify
import base64
import zipfile
import tempfile
import re
from os import unlink

cap = pyshark.FileCapture("phreaky.pcap", display_filter="imf")
output = "phreaky.pdf" 

def b64unzipper(output, b64, password):
    with tempfile.NamedTemporaryFile(delete=False) as tmp:
        tmpPath = tmp.name
    with open(tmpPath, "wb") as zz:
        zz.write(b64)
    try:

        with zipfile.ZipFile(tmpPath, "r") as zf:
            for filename in zf.namelist():
                with zf.open(filename, pwd=password.encode()) as zb:
                    Contents = zb.read()

                with open(output, "ab") as out:
                    out.write(Contents)
    except zipfile.BadZipFile:
        print("BADZIP")
    finally:
        unlink(tmpPath)

def PasswordLookup(pattern, string):
    match = re.search(pattern, string)
    if match:
        password = match.group(1)
        return str(password)


for pkt in cap:

    la = pkt.layers[4] #selecting imf layer
    media = la.media_type   #base64 string header
    passContainer = str(la.get_field_value(''))
    pattern = r"Password:\s*([^\s\\]+)"   #regex pattern enumerating zip passwords in passCOntainer
    
    password = PasswordLookup(pattern, passContainer)

    counter = 0
    arr = []
    for byte in media:
        if counter == 2:
            counter = 0
            pass
        else:
            counter += 1
            arr.append(byte)
    payload_0 = unhexlify(''.join(arr)).decode('utf-8')
    payload_1 = payload_0.replace("\r\n","")
    payload_decoded = base64.b64decode(payload_1)
    b64unzipper(output, payload_decoded, password)

print('Successfully Extracted file at ', output)

```
we run it and boom we have the pdf file containing the flag.  

![](https://github.com/z4rr1t/writeups/blob/main/HTB/CA2024%3AHR/Phreaky/img/phreakingPDF.png?raw=true)

## flag
```HTB{Th3Phr3aksReadyT0Att4ck}```

