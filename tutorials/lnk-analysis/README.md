# LNK Malware Analysis with PE Explorer+

In recent years, malicious campaigns leveraging Windows shortcut (LNK) files have increased significantly. Despite their growing use in real-world attacks, awareness of malicious LNK files remains relatively limited, making them an effective attack vector.

Unlike traditional executable files or malicious documents, LNK files can employ techniques that are easily overlooked by conventional detection mechanisms. As a result, threat actors increasingly use them as an initial access vector.

Palo Alto Networks Unit 42 conducted an in-depth analysis of approximately 30,000 malicious LNK files and published [their findings](https://unit42.paloaltonetworks.com/lnk-malware/).

Based on the techniques discussed in that research, this tutorial demonstrates how to analyze nine **in-the-wild** malicious LNK samples using PE Explorer+.

- **(Type 1) In-argument Scripts**
  - Case 1: Command Prompt
  - Case 2: Conhost
  - Case 3: Forfiles
- **(Type 2) Overlay**
  - Case 4: Find / Findstr
  - Case 5: Mshta
  - Case 6: PowerShell Commands
- **(Type 3) Exploit**
  - Case 7: CVE-2010-2568
  - Case 8: CVE-2017-8464
  - Case 9: CVE-2025-9491

---

## Case 1: Command Prompt

![Case 1-1](images/Case-1/fig-1.png)

Let's start with the first sample in the **In-argument Scripts** category.

We can see that it uses `cmd.exe` to execute encoded PowerShell code. PE Explorer+ automatically decodes the code, revealing that the LNK file downloads a DLL from GitHub and loads it.

---

## Case 2: Conhost

![Case 2-1](images/Case-2/fig-1.png)

The second sample uses `conhost.exe`.

Since the **Arguments** field contains encoded content, we need to decode it. Let's use CyberChef.

![Case 2-2](images/Case-2/fig-2.png)

Double-clicking the **Arguments** field copies its contents to the clipboard.

Using `Unescape string` in CyberChef produces a clean, decoded representation of the command.

The decoded command creates a directory at a specified path, writes a JavaScript file to it, and executes the file. The URL used to retrieve the payload can also be identified directly from the decoded command.

---

## Case 3: Forfiles

![Case 3-1](images/Case-3/fig-1.png)

The third sample uses `forfiles`.

In `forfiles`, the `/p` option specifies the path to search, while `/c` specifies the command to execute for matching files.

The command uses wildcards, likely as an attempt to evade simple signature-based detection. Ultimately, it invokes `C:\Windows\System32\mshta.exe` to retrieve and execute a remote HTA file.

---

## Case 4: Find / Findstr

![Case 4-1](images/Case-4/fig-1.png)

Now let's move on to the **Overlay** category.

The behavior specified in the **Arguments** field can be summarized as follows:

1. Search the current LNK file for content matching a specific marker (`CiRFcnJvckFjdGlvbl`) and save the result to `%tmp%\Temp.jpg`.
2. Base64-decode `Temp.jpg` and execute the decoded content with PowerShell.

Arbitrary data can be appended to the end of an LNK file without interfering with normal parsing and processing of the LNK structure. As a result, embedded payloads are often found in the overlay at the end of the file.

Click the `Go to overlay` button.

![Case 4-2](images/Case-4/fig-2.png)

We can see a string that appears to be Base64-encoded, but it does not begin with the marker we are looking for.

Select `Strings` from the tree view on the left.

![Case 4-3](images/Case-4/fig-3.png)

The marker can be found in the last line.

This suggests that the sample contains two separate encoded data blobs.

First, copy the encoded string referenced by the **Arguments** field and decode it.

![Case 4-4](images/Case-4/fig-4.png)

After decoding it from Base64 in CyberChef, we can use the syntax highlighter to make the resulting code easier to inspect.

![Case 4-5](images/Case-4/fig-5.png)

The relevant portion of the decoded code is shown above.

The first Base64-encoded blob we identified turns out to be a decoy PDF document.

The malware then sends system information to its C2 server at `pdf-online[.]top` and performs additional actions based on commands received from the server.

---

## Case 5: Mshta

![Case 5-1](images/Case-5/fig-1.png)

In this case, `mshta` is executed with the current LNK file itself passed as an argument.

Unlike the previous case, this sample does not rely on a marker to locate the payload.

`mshta` is relatively permissive when parsing input and can identify and execute embedded HTA content within a file.

Click the `Go to overlay` button.

![Case 5-2](images/Case-5/fig-2.png)

We can see HTA code appended after the end of the LNK structure.

From the **Summary** view, click `Save overlay...` to extract the overlay to a separate file.

![Case 5-3](images/Case-5/fig-3.png)

The script decodes the PowerShell code stored in the `power` variable, writes the decoded content to:

`C:\Users\Public\PublicDocuments\ServiceRs.ps1`

and then executes it.

![Case 5-4](images/Case-5/fig-4.png)

Decoding the PowerShell code stored in the `power` variable reveals the final payload. The payload implements additional functionality, including persistence mechanisms, but further analysis is outside the scope of this tutorial.

---

## Case 6: PowerShell Commands

![Case 6-1](images/Case-6/fig-1.png)

This sample uses PowerShell commands to locate a marker (`BS:D`) within the LNK file, extract the data following the marker, save it as an executable, and run it.

Click `Go to overlay`.

![Case 6-2](images/Case-6/fig-2.png)

The `BS:D` marker is immediately visible.

Copying and decoding the data following the marker reveals that the embedded payload is a PE file. We will skip further analysis of this PE file.

---

## Case 7: CVE-2010-2568

![Case 7-1](images/Case-7/fig-1.png)

Now let's take a look at the **Exploit** category.

This sample exploits CVE-2010-2568, the infamous Windows Shell vulnerability used by `Stuxnet` in 2010.

Click the vulnerability information entry displayed at the bottom of the window.

![Case 7-2](images/Case-7/fig-2.png)

CVE-2010-2568 occurs when Windows Shell processes the Shell Item IDList to display an LNK file's icon and improperly trusts a path referencing an external CPL/DLL.

As a result, the attacker-specified CPL/DLL can be loaded and arbitrary code executed merely when Explorer renders the shortcut's icon, without requiring the user to explicitly launch the LNK file.

In this sample, the `.CPL` file referenced by the LNK is ultimately loaded and executed automatically.

---

## Case 8: CVE-2017-8464

![Case 8-1](images/Case-8/fig-1.png)

At the bottom of the window, PE Explorer+ identifies two vulnerabilities: CVE-2010-2568 and CVE-2017-8464.

CVE-2017-8464 is closely related to CVE-2010-2568 and involves a similar LNK-based code execution technique after protections introduced for the earlier vulnerability proved insufficient against certain crafted shortcuts.

![Case 8-2](images/Case-8/fig-2.png)

The **Shell Item Type** and several other properties closely resemble those observed in the previous exploit sample.

One important difference is the use of a `SpecialFolderDataBlock`. This structure is used to reference the Control Panel namespace as part of the technique used to reach the attacker-controlled CPL/DLL.

![Case 8-3](images/Case-8/fig-3.png)

The **SpecialFolderDataBlock** references the `Control Panel`, while **FirstChildSegmentOffset** points to `ItemID[2]`.

Through this specially crafted LNK structure, the attacker-controlled CPL/DLL can ultimately be loaded, resulting in arbitrary code execution.

---

## Case 9: CVE-2025-9491

![Case 9-1](images/Case-9/fig-1.png)

The final case covers a relatively recently disclosed vulnerability.

CVE-2025-9491 is a **UI misrepresentation vulnerability** involving how Windows displays command-line arguments associated with LNK files.

An attacker can insert a large number of spaces and control characters into the `Arguments` field so that the actual malicious command appearing later in the command line is not visible in the Target field shown by the Windows properties UI.

Unlike CVE-2010-2568 and CVE-2017-8464, this technique does not cause a DLL or CPL to be automatically loaded while Explorer processes the shortcut icon. Instead, the victim must execute the crafted LNK file.

The key issue is that the malicious arguments can be obscured in the Windows UI, making it difficult for the user to identify the command that will actually be executed.

In the **Summary** tab, click the **Suspicious** entry displayed at the bottom.

![Case 9-2](images/Case-9/fig-2.png)

PE Explorer+ detects the padding characters and displays the actual command as the **Decoded value**.

Double-click the `File offset` value (`2E9`).

![Case 9-3](images/Case-9/fig-3.png)

The hex view shows that a large number of spaces and control characters appear before the PowerShell command, demonstrating how the actual command is obscured from the normal Windows properties UI.

---

## Conclusion

In this tutorial, we analyzed nine **real-world malicious LNK samples** using PE Explorer+.

Unlike traditional executable malware or malicious documents, LNK files are still often perceived as relatively harmless shortcuts. Attackers can take advantage of this perception, and malicious LNK files continue to be used as an initial access vector in real-world campaigns.

Although tools for inspecting LNK structures already exist, many of them are primarily command-line based. PE Explorer+ provides a GUI-based approach intended to make structural inspection and malware analysis of LNK files more convenient for analysts.

Hopefully, PE Explorer+ can be a useful addition to your toolkit when analyzing and investigating malicious LNK files. ;)