# Crackme writeup - git's obfuscated password
## Introduction
- **Source:** Crackmes.one  
- **Link:** https://crackmes.one/crackme/68b82fc68fac2855fe6fbaa8
- **Challange name:** Obfuscated password  
- **Task:** Find the obfuscated password. 
- **Used tools:** Strings, Ghidra  

---

### Step 1: Basic string search  
The first thing i did after i downloaded the binary to my vm was a basic string search: `strings crackme.exe`  

**Output:**
```bash
bad allocation
Unknown exception
bad array new length
string too long
bad cast
VG90YWxseU5vdFRoZVBhc3N3b3Jk
Greetings User1...
Please enter your Password: 
Login successful!
Press Enter to continue...
Invalid password. Access denied.
RSDS
"sxG
...
```

So it was clear that the password would be requested via user input. Therefore, I should look for the strings "Please enter your Password:" and "Login successful!" in the decompiled binaray.  

---

### Step 2: Open the binary in ghidra  
After I created a new project and analyzed the binary with Ghidra, the entry point showed two functions:   
```c++
  FUN_140002ad0();
  FUN_140002534();
```  
The code inside these functions was not immediately obvious to me. After a bit of research I learned that the first function is usually part of the C runtime startup (CRT). So there isn’t much of interest there besides initialization (heap, locale, security cookie, exception handling, etc.).

The other function on the other hand is usually where the main function gets called. I inspected its code and searched for argc/argv. In the decompiled section I found:  

```c++
      uVar7 = _get_initial_narrow_environment();
      puVar8 = (undefined8 *)__p___argv();
      uVar4 = *puVar8;
      puVar9 = (uint *)__p___argc();
      uVar6 = (ulonglong)*puVar9;
      iVar3 = FUN_140001820(uVar6,uVar4,uVar7);
```
So the function `FUN_140001820` receives three parameters: argc (uVar6), argv (uVar4), and uVar7. That looks like the main. After following references to that function I found the strings I was searching for during my initial string scan, so I was on the right track.  

---

### Step 3: Analysing the main  
When I first opened the main function of this program, I noticed the user input request:
```c++
  FUN_140001c40((basic_ostream<> *)cout_exref,"Greetings User1...");
  FUN_140001c40((basic_ostream<> *)cout_exref,"\nPlease enter your Password: ");
  FUN_140001e20((basic_istream<> *)cin_exref,(longlong *)&local_38,param_3);
  FUN_1400012a0(local_58,(byte *)&local_b8);
```  
The most interesting part here was the call to the function FUN_1400012a0. This function receives two arguments: local_58 and a pointer to local_b8. local_58 looks like a destination/container (probably a std::string object or a buffer). By looking at the code of FUN_1400012a0, I noticed that the first parameter is returned, which supports my theory
  
Later in the main function, local_58 is assigned to _Buf2. This variable is then used in a byte-wise comparison:
```c++
  if (local_28 == local_48) {
    if (local_28 != 0) {
      iVar4 = memcmp(ppppuVar5,_Buf2,local_28);
      if (iVar4 != 0) goto LAB_140001988;
    }
    FUN_140001c40((basic_ostream<> *)cout_exref,"Login successful!");
  }
  else {
LAB_140001988:
    this = FUN_140001c40((basic_ostream<> *)cout_exref,"Invalid password. Access denied.");
    std::basic_ostream<>::operator<<(this,FUN_140002000);
  }
```
So it’s clear that local_58 contains the password I was looking for, and this variable is filled in the function FUN_1400012a0. However, there is another argument to consider: local_b8. Unlike local_58, which is initialized as an empty array: `longlong local_58 [2];` the variable `local_b8` already contains values:
```c++
  *local_b8 = 0x7378575930394756;
  local_b8[1] = 0x6f52466476355565;
  local_b8[2] = 0x334e33636842565a;
  *(undefined4 *)(local_b8 + 3) = 0x6b4a3362;
  *(undefined *)((longlong)local_b8 + 0x1c) = 0;
```  
So local_b8 is an array of some integer type because i learnd a hex literal without a suffix (0x1234...) is initially treated as an integer literal in C/C++. The last value sets a byte at offset 0x1C (28 decimal) to 0 — this is the null terminator (\0). So local_b8 is a string.  
After converting the hex bytes `73 78 57 59 30 39 47 56` into ASCII, I got: `sxWY09GV`.

Since I am on an x86/x64 architecture, the byte representation is little-endian, so I had to reverse the output: `VG90YWxs`.   
At first, I suspected this was some kind of encrypted word. After a bit of research, I found out that it was Base64-encoded. After decoding it, I got the word: `Totally`.  
So i decoded all of the values with a simple bash command: `echo "VG90YWxseU5vdFRoZVBhc3N3b3Jk" | base64 -d`  
And the result was: `TotallyNotThePassword`.  
  
I quickly realized that this was the same string I had found in my very first string scan with the String utility.  
So this challenge could have been solved with a basic string search and a simple bas64 decoding

I ran the programm in my Win VM, typed in the password and got the message: "Login successful!".
