# CyLab CTF: Warm

##  Challenge
This program has extraordinarily helpful information. Download the file and find the flag.

##  Walkthrough
1. Used `wget` to download the file directly into Kali:
   `wget https://challenge-files.cylabacademy.net/.../warm`
2. Made the file executable:
   `chmod +x warm`
3. Tried the standard help flag, which failed:
   `./warm --help` -> Output: "I don't know what '--help' means! I do know what -h means though!"
4. Ran it with the `-h` flag:
   `./warm -h`
5. The program printed the flag.

##  Flag
`academy{b1scu1ts_4nd_gr4vy_57d8c79}`
