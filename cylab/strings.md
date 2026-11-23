# CyLab CTF: Strings

##  Challenge
Find the flag hidden inside a binary file without executing it.

##  Walkthrough
1. Downloaded the binary file.
2. Used the `strings` command to extract human-readable text.
3. Piped the output to `grep` to search for the flag:
   ```bash
   strings strings | grep -i "academy"
   
4. The flag appeared in the output.

Flag
academy{5tRIng5_1T_DAEbE8bd}
