Overview

This room covers John the Ripper (JtR), specifically the Jumbo John community edition, and walks through cracking a range of hash and file types:


Raw hashes (MD5, SHA1, SHA256, SHA512, Whirlpool, etc.)
Linux /etc/passwd + /etc/shadow combos (unshadowing)
Single crack mode
Custom mangling rules
Password-protected ZIP files
Password-protected RAR files
SSH private key (id_rsa) passphrases


The core idea behind every task is the same: John takes a hash (or a value derived from one), runs candidate passwords from a wordlist or ruleset through the same hashing algorithm, and checks for a match. This is a dictionary/brute-force attack, not a cryptographic break of the hash itself.

Setup


Used the AttackBox / attached VM, so John was pre-installed — no need to build from source.
Wordlist: /usr/share/wordlists/rockyou.txt (a leaked password list, ~14 million entries).
If not already unzipped: tar xvzf rockyou.txt.tar.gz.


Task 1–3: Background & Terminology

Conceptual tasks covering what hashing is, why it's one-way, and why dictionary attacks work despite that. No commands here — just definitions of hashing, salting, and the difference between JtR and Jumbo John.

Task 4: Cracking Basic Hashes

General syntax:

bashjohn --format=<hash_type> --wordlist=/usr/share/wordlists/rockyou.txt <hash_file>

To view a cracked password once John has it in its pot file:

bashjohn --show --format=<hash_type> <hash_file>

Notes:


Standard hash types need a raw- prefix (e.g. raw-md5, raw-sha1, raw-sha256) unless they come from a known application format (e.g. nt for Windows NTLM).
If you don't know the hash type, hash-identifier or name-that-hash can help, or you can just try --format=raw-<guess> and see if John accepts it.
To list every format John supports: john --list=formats, optionally piped through grep.


Task 5: Cracking Windows NTLM Hashes

NTLM hashes use the nt format (no raw- prefix needed):

bashjohn --format=nt --wordlist=/usr/share/wordlists/rockyou.txt ntlm.txt

Task 6: Cracking Linux /etc/shadow Hashes

Linux systems split user info across /etc/passwd (usernames, UID/GID, etc.) and /etc/shadow (password hashes). To crack these with John you first need to recombine ("unshadow") them:

bashunshadow local_passwd local_shadow > unshadowed.txt
john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed.txt

unshadow merges the two files into a single /etc/passwd-style format that John can parse natively (it auto-detects the hash type, usually SHA-512 crypt on modern systems).

Task 7: Single Crack Mode

Single crack mode uses info about the user (username, GECOS fields) to generate likely password candidates — useful when you suspect the password is related to the username, e.g. Joker1! for a user called joker.

bashjohn --single --format=<hash_type> hash_file.txt

Format for the input file matters here — John expects username:hash on one line, so you may need to prepend the username manually before running.

Task 8: Custom Rules

Custom mangling rules live in john.conf (typically /opt/john/john.conf on the AttackBox, or /etc/john/john.conf elsewhere). You define a rule set under a header like:

[List.Rules:MyRules]
Az"[0-9][0-9][0-9]"
A0"!"


Az"..." appends the given characters to the end of each candidate word.
A0"..." prepends characters to the front.


Then invoke it with:

bashjohn --wordlist=/usr/share/wordlists/rockyou.txt --rules=MyRules --format=<hash_type> hash_file.txt

This is useful when you know something about password structure (e.g. "probably ends in a digit and a special character") but don't want to brute-force blindly.

Task 9: Cracking ZIP File Passwords

Two-step process — extract a crackable hash from the archive, then crack it:

bashzip2john secure.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
john --show zip.hash

Task 10: Cracking RAR File Passwords

Same pattern, different extraction tool:

bashrar2john secure.rar > rar.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash
john --show rar.hash

Task 11: Cracking SSH Private Key Passphrases

If an id_rsa file is passphrase-protected, ssh2john converts it into a crackable hash:

bashpython3 /opt/john/ssh2john.py id_rsa > ssh.hash
john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
john --show ssh.hash


Key Takeaways


JtR's workflow is always: get a hash → identify format → choose an attack mode → run against a wordlist/ruleset.
The *2john family of tools (zip2john, rar2john, ssh2john, and others like office2john, pdf2john) exist specifically to convert proprietary file formats into a hash John can process.
Wordlist quality matters more than raw compute for weak/common passwords — rockyou.txt alone cracks a surprising number of real-world passwords because it's built from actual breached credentials.
Single crack mode + custom rules are far more efficient than blind wordlist attacks when you have context about the target (username conventions, password policies, etc.) — this mirrors real-world password auditing during pentests.
Always double check hash format before cracking — a wrong --format flag is the most common reason a crack silently fails or takes forever.
