# Horace's payload - RH CTF 2022

The challenge presents a `horace.o` file and the following text:

> We captured a piece of software from one of Mr. Odeur's agents on
> their way back from the meeting point after their attack on the rubber
> ducky factory. Our monitoring indicates some kind of issue with the
> factory's production lines usually reachable at
> `factory.ctf.nofizz.buzz`. We haven't been able to decode what it
> is. Please examine the package and decode it.

Running `file` on the file horace.o gives some hints on the content:

```
$ file horace.o 
horace.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), not stripped
```

and `nm` indicates that the object file contains these symbols:

```
0000000000000000 T cls_main
0000000000000148 T compute_checksum
0000000000000000 D key_map
0000000000000138 t LBB0_2
0000000000000190 t LBB1_2
00000000000001d8 t LBB1_3
0000000000000068 t LBB2_1
0000000000000268 t LBB2_10
0000000000000420 t LBB2_20
0000000000000460 t LBB2_21
0000000000000468 t LBB2_22
0000000000000188 t LBB2_23
0000000000000048 t LBB2_3
0000000000000000 D __license
0000000000000000 T read_key_chunk_from_map
```

As confirmed by the `tc-bpf` man page, the file seems to contain a BPF
program that can be attached to the ingress queue of an interface to
filter packets. 

Let's disassemble it with objdump and check what it does.

Starting from the entry point `cls_main()`, the first part of the
program repeatedly calls `bpf_map_lookup_elem()` for 42 times and
stores the result into a buffer located at `fp - 232`, while logging
some messages through `bpf_trace_printk()`:

```asm
0000000000000000 <cls_main>:
   0:   stxdw [%fp+-240],%r1               ; save context (r1) into [fp-240]
   8:   mov %r9,%fp                        
  10:   add %r9,-232                       ; r9 = ptr to buffer for the values retrieved from the map
  18:   lddw %r6,0x5f70616d5f667062        ; "_pam_fpb"
  20:   
  28:   lddw %r8,0x20676e696c6c6143        ; " gnillaC"
  30:   
  38:   mov %r7,0                          ; r7 = counter (from 0 to 41)
  40:   ja 4                               ; goto 68

0000000000000048 <LBB2_3>:
  48:   stxw [%r9+0],%r1                   ; *ptr = r1
  50:   add %r9,4                          ; ptr += 4
  58:   add %r7,1                          ; r7++
  60:   jeq %r7,0x2a,36                    ; if r7 == 42 jump at 0x188

0000000000000068 <LBB2_1>:
  68:   stxw [%fp+-4],%r7                  
  70:   mov %r1,0                          
  78:   stxb [%fp+-10],%r1                 ; fp - 10 = "\0"
  80:   mov %r1,0xa64                      
  88:   stxh [%fp+-12],%r1                 ; fp = 12 = "d\n"        
  90:   mov %r1,0x25203d20                 
  98:   stxw [%fp+-16],%r1                 ; fp - 16 = " = %"
  a0:   lddw %r1,0x7864692074612065        
  a8:   
  b0:   stxdw [%fp+-24],%r1                ; fp - 24 = "xdi ta e"
  b8:   lddw %r1,0x756c617620646165        
  c0:   
  c8:   stxdw [%fp+-32],%r1                ; fp - 32 = "ulav dae"
  d0:   lddw %r1,0x72206f74206d656c        
  d8:   
  e0:   stxdw [%fp+-40],%r1                ; fp - 40 = "r ot mel"
  e8:   lddw %r1,0x655f70756b6f6f6c        
  f0:   
  f8:   stxdw [%fp+-48],%r1                ; fp - 48 = "e_pukool"
 100:   stxdw [%fp+-56],%r6                ; fp - 56 = "_pam_fpb"
 108:   stxdw [%fp+-64],%r8                ; fp - 64 = " gnillaC"

; frame now contains                                              FP
;    -6         -5       -4        -3        -2        -1         0
; 43210987654321098765432109876543210987654321098765432109876543210
; Calling bpf_map_lookup_elem to read value at idx = %dN0

 110:   mov %r1,%fp                       ; 
 118:   add %r1,-64                       ; arg1 = format string (fp - 64)
 120:   mov %r2,0x37                      ; arg2 = len
 128:   mov %r3,%r7                       ; arg3 = counter
 130:   call 6                            ; Call helper bpf_trace_printk()
 138:   mov %r2,%fp                       
 140:   add %r2,-4                        ; arg2 = fp -4
 148:   lddw %r1,0                        ; arg1 = 0
 150:                                     ; (arg3 = counter)
 158:   call 1                            ; Call helper bpf_map_lookup_elem(map, key)
 160:   mov %r1,0xaa                      ; \   
 168:   jeq %r0,0,-37                     ;  |
 170:   ldxw %r1,[%r0+0]                  ;  -> set in r1 the value from map or 0xaa and goto 0x48
 178:   and %r1,0xff                      ;  |
 180:   ja -40                            ; /
```

For now the purpose of the buffer doesn't seem very clear, but it will
be later.

The following part of the program contains some checks. I didn't know
what the offsets (0x50, 0x4c) referred to, but looking at the
comparison with magic numbers as 14 (ethernet header length) and
0x0608 (ARP EtherType), it is clear that `r1` is a pointer to the
packet, and `r2` is probably the end. The program sets `r0` to zero
(and jumps to the end) to accept the packet, or sets it to 2 to drop
it.

```asm
 188:   ldxdw %r1,[%fp+-240]              ; r1 = context (skb?)
 190:   ldxw %r2,[%r1+0x50]               ; r2 = packet end
 198:   ldxw %r1,[%r1+0x4c]               ; r1 = packet start
 1a0:   mov %r3,%r1                       
 1a8:   add %r3,0xe                       
 1b0:   mov %r0,2                         
 1b8:   jgt %r3,%r2,85                    ; if packet is shorter than 14, return 2 (DROP)
 1c0:   ldxb %r4,[%r1+0xc]                
 1c8:   ldxb %r3,[%r1+0xd]                
 1d0:   lsh %r3,8                         
 1d8:   or %r3,%r4                        ; r3 now contains the EtherType of the frame
 1e0:   mov %r0,0                         
 1e8:   jeq %r3,0x608,79                  ; if EtherType == ARP => ACCEPT
 1f0:   jne %r3,8,77                      ; if EtherType != IP => DROP
 1f8:   mov %r3,%r1                       
 200:   add %r3,0x54                      
 208:   mov %r0,2                         
 210:   jgt %r3,%r2,74                    ; if lenght < 84 => DROP
 218:   ldxb %r2,[%r1+0x17]               
 220:   mov %r0,2                         
 228:   jne %r2,0x11,71                   ; if IP.Protocol != 17 (UDP) => DROP
 230:   ldxh %r2,[%r1+0x24]               
 238:   mov %r0,2                         
 240:   jne %r2,0xf27,68                  ; if UDP.dport != 9999 => DROP
 248:   mov %r2,%r1                       
 250:   add %r2,0x2a                      ; r2 = packet + 42
 258:   mov %r4,0                         
 260:   mov %r5,0                         

```

Then we have some code to sum all the bytes of the UDP payload:

```asm
 268:   mov %r3,%r2                       
 270:   add %r3,%r4                       
 278:   ldxb %r0,[%r3+0]                  
 280:   mov %r3,%r5                       
 288:   add %r3,%r0                       
 290:   mov %r5,%r3                       
 298:   and %r5,0xff                      
 2a0:   add %r4,1                         
 2a8:   jne %r4,0x2a,-9                   ; if r4 != 0x2a goto 0x268
```

Assuming `base = r2`, `i = r4`, `sum = r5`, this is equivalent to:

```C
base = packet + sizeof(EthHeader) sizeof(IPHeader) + sizeof(UDPHeader);
sum = 0;
for (i = 0; i < 42; i++) {
    sum += base[i];
    sum &= 0xff;
}
```

Then we have a series of checks on the content of the UDP payload:

```asm
 2b0:   ldxb %r4,[%r1+0x2d]               ; r4 = payload[3]
 2b8:   ldxb %r5,[%r1+0x2a]               ; r5 = payload[0]
 2c0:   mov %r0,2
 2c8:   jne %r5,%r4,51                    ; if payload[0] != payload[3] => DROP
 2d0:   ldxb %r4,[%r1+0x2b]               ; r4 = payload[1]
 2d8:   ldxb %r5,[%r1+0x2c]               ; r5 = payload[2]
 2e0:   add %r5,%r4                       
 2e8:   mov %r0,2
 2f0:   jne %r5,8,46                      ; if payload[1] + payload[2] != 8 => DROP
 ...
 ...                                      ; if payload[10] != payload[9] * 2 => DROP
 ...                                      ; if payload[13] != payload[11] ^ payload[12] => DROP
 ...                                      ; ...
```

Finally, we have a check on the sum of the bytes of the payload
computed before:

```asm
 3e8:   and %r3,0xff                      
 3f0:   mov %r0,2
 3f8:   jne %r3,0xe4,13                   ; if sum(payload[]) != 0xe4 => DROP
```

Now the packet is validated. The BPF program overwrites the packet
with the data from the buffer at `fp-232` that was populated by
reading the map.

```asm
 400:   mov %r0,0                         
 408:   mov %r1,%fp                       
 410:   add %r1,-232                      ; r1 = fp - 232
 418:   mov %r3,0                         ; r3 = 0 (i = 0)
 420:   mov %r4,%r2                       ; 
 428:   add %r4,%r3                       ; r4 = UDP payload + i
 430:   ldxw %r5,[%r1+0]                  ;
 438:   stxb [%r4+0],%r5                  ; *r4 = *r1
 440:   add %r1,4                         ; r1 = r1 + 4
 448:   add %r3,1                         ; r3++ (i++)
 450:   jeq %r3,0x2a,2                    if r3 == 42 =>  ACCEPT
 458:   ja -8                             goto 0x420

0000000000000460 <LBB2_21>:
 460:   mov %r0,2                         ; DROP

0000000000000468 <LBB2_22>:
 468:   exit
```

---

We can now try to send to server `factory.ctf.nofizz.buzz` a UDP
packet on port 9999 with a content that passes the filter, and
hopefully we'll get something back.

Let's start tcpdump to analyze the traffic:

```
$ sudo tcpdump -X -i any udp port 9999
```

The following program creates a UDP packet with the 42-bytes payload
and sends it:

```python
#!/bin/python3

from scapy.all import *

payload=[0 for i in range(42)]

# p[0] = p[3]
payload[0] = 1
payload[3] = 1

# p[1] + p[2] = 8
payload[1] = 3
payload[2] = 5

# p[5] / p[4] = 1
payload[5] = 9
payload[4] = 9

# p[6] - p[7] - p[8] = 3
payload[6] = 12
payload[7] = 6
payload[8] = 3

# p[10] = p[9] * 2
payload[9] = 5
payload[10] = 10

# p[13] = p[11] ^ p[12]
payload[11] = 0
payload[12] = 0
payload[13] = 0

# p[14] & p[15] = 0xff
payload[14] = 255
payload[15] = 255

# (sum of all bytes of p[]) & 0xff = 0xe4
s = sum(payload)
payload[16] = (0xe4 - s) & 0xff

payload=bytearray(payload)

send(IP(dst="10.0.108.144") / UDP(sport=RandShort(), dport=9999) / Raw(load=payload))

```

After running it, tcpdump shows:

```
17:46:13.162420 tun9  Out IP tp.20445 > 10.0.108.144.distinct: UDP, length 42
	0x0000:  4500 0046 0001 0000 4011 390c 0a28 c0e2  E..F....@.9..(..
	0x0010:  0a00 6c90 4fdd 270f 0032 78ea 0103 0501  ..l.O.'..2x.....
	0x0020:  0909 0c06 0305 0a00 0000 ffff a600 0000  ................
	0x0030:  0000 0000 0000 0000 0000 0000 0000 0000  ................
	0x0040:  0000 0000 0000                           ......
17:46:13.323616 tun9  In  IP 10.0.108.144.distinct > tp.20445: UDP, length 42
	0x0000:  4500 0046 4e55 4000 3411 b6b7 0a00 6c90  E..FNU@.4.....l.
	0x0010:  0a28 c0e2 270f 4fdd 0032 cbbb 5248 5f43  .(..'.O..2..RH_C
	0x0020:  5446 7b52 4553 5441 5254 5f43 4f44 455f  TF{RESTART_CODE_
	0x0030:  3039 3832 335f 6542 5066 5f5f 4637 575f  09823_eBPf__F7W_
	0x0040:  5f39 3833 347d                           _9834}
```

That looks like a flag!
