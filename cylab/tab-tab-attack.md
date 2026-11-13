# CyLab CTF: Tab Tab Attack

##  Challenge
Using tab complete in the Terminal will add years to your life, esp. when dealing with long rambling directory structures and filenames.

##  Walkthrough
1. Downloaded the zip file using `wget`:
   `wget https://challenge-files.cylabacademy.net/.../Addadshashanammu.zip`
2. Unzipped the file:
   `unzip Addadshashanammu.zip`
3. Navigated through the massive directory structure using `cd`:
   `cd Addadshashanammu/Almurbalarammi/Ashalmimilkala/...`
4. Found a binary file named `fang-of-haynekhtnamet`.
5. Made the binary executable:
   `chmod +x fang-of-haynekhtnamet`
6. Executed the binary:
   `./fang-of-haynekhtnamet`
7. The program printed the flag.

##  Flag
`academy{l3v3l_up!_t4k3_4_r35t!_9d928112}`
