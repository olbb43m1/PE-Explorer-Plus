# HWP Malware Analysis with PE Explorer+

Let's take a look at how quickly and efficiently malicious HWP files can be analyzed using **PE Explorer+**.

For this tutorial, we prepared four in-the-wild samples representing different types of HWP-based attacks:

- Macro
- EPS
- OLE
- Exploit

Each case demonstrates how PE Explorer+ can help identify suspicious content and trace it down to the final payload.

---

## Case 1: Macro

![Case 1 - Macro](hwp-analysis/images/Case-1-Macro/fig-1.png)

When the file is opened, PE Explorer+ displays basic document information along with potentially suspicious items.

A suspicious script has been detected in this sample. Let's follow the link to take a closer look.

![Case 1 - Macro Script](hwp-analysis/images/Case-1-Macro/fig-2.png)

Following the link takes us directly to the decompressed stream, where its contents can be inspected as text.

A quick look at the script reveals that it attempts to execute an encoded payload using PowerShell.

![Case 1 - Decoded Payload](hwp-analysis/images/Case-1-Macro/fig-3.png)

We can copy the encoded data into **CyberChef** and decode it.

The result reveals the final payload—in this case, PowerShell code—that the malicious HWP document attempts to execute.

---

## Case 2: EPS

![Case 2 - EPS](hwp-analysis/images/Case-2-EPS/fig-1.png)

Next, let's examine a sample that uses an **EPS (Encapsulated PostScript)** exploit technique, which was once widely used in malicious HWP campaigns.

These attacks typically embed a crafted EPS/PostScript object in the HWP document and exploit vulnerabilities in external components used to process or render the EPS content.

PE Explorer+ flags a file under `BinData` as suspicious. Let's follow the link and inspect it.

![Case 2 - EPS Stream](hwp-analysis/images/Case-2-EPS/fig-2.png)

In the decompressed stream, we can see PostScript code along with an encoded payload.

Looking more closely at the script reveals that the payload is XOR-decoded using a 4-byte key.

Let's decode the payload using **CyberChef**.

![Case 2 - EPS Payload](hwp-analysis/images/Case-2-EPS/fig-3.png)

The decoded data reveals the final payload—shellcode—intended to be executed as part of the exploitation chain involving the external PostScript processing component.

---

## Case 3: OLE

![Case 3 - OLE](hwp-analysis/images/Case-3-OLE/fig-1.png)

Now let's examine a malicious document containing an embedded **OLE object**, a technique that continues to appear in malicious documents.

Click the `BinData → BIN0002.OLE` link to inspect the embedded object.

![Case 3 - OLE Signature](hwp-analysis/images/Case-3-OLE/fig-2.png)

The decompressed stream shows the familiar OLE/CFB signature:

`D0 CF 11 E0 A1 B1 1A E1`

PE Explorer+ can load the embedded OLE object directly using the **Open as new tab** button.

![Case 3 - OLE Streams](hwp-analysis/images/Case-3-OLE/fig-3.png)

The OLE object contains two streams:

- `CompObj`
- `Ole10Native`

![Case 3 - Ole10Native](hwp-analysis/images/Case-3-OLE/fig-4.png)

Opening the `Ole10Native` stream reveals a file path followed by PE data.

We could extract the PE data manually using the dump functionality, but PE Explorer+ provides an easier way to access it through the **Embedded Files** feature.

![Case 3 - Embedded Files](hwp-analysis/images/Case-3-OLE/fig-5.png)

Under **Embedded Files**, click **Open as new tab** to load the detected PE file directly.

![Case 3 - Final PE Payload](hwp-analysis/images/Case-3-OLE/fig-6.png)

We have now reached the final payload: the PE file embedded inside the OLE object.

---

## Case 4: Exploit

![Case 4 - Exploit](hwp-analysis/images/Case-4-Exploit/fig-1.png)

Finally, let's look at an exploit targeting a vulnerability in the HWP application itself.

When the sample is opened, PE Explorer+ identifies suspicious records across multiple sections of the document.

Let's select one of them for further analysis.

![Case 4 - Suspicious Record](hwp-analysis/images/Case-4-Exploit/fig-2.png)

One of the `Paragraph Text` records is unusually large—approximately **18 MB**.

Looking at the raw data at the bottom of the window, we can immediately notice a highly repetitive pattern.

Double-clicking the **Offset** field jumps directly to the corresponding location in the **Hex (decomp)** tab.

> **Note**
>
> This sample is a *Distribution Document*. HWP distribution documents store their data streams in encrypted form to provide read-only protection.
>
> For analysis, PE Explorer+ identifies the required encryption key and automatically decrypts and decompresses the protected streams.

![Case 4 - NOP Sled](hwp-analysis/images/Case-4-Exploit/fig-3.png)

The stream contains a large amount of repetitive data.

This pattern forms a **NOP sled**, which increases the likelihood that execution redirected into the affected memory region will eventually reach the intended payload after successful exploitation.

![Case 4 - Shellcode](hwp-analysis/images/Case-4-Exploit/fig-4.png)

Following the NOP sled, we can identify the shellcode that is intended to execute as part of the exploit.

The decompressed data can be saved to a file using the **Dump (decomp)** feature for further analysis with other tools.

---

## Conclusion

In this tutorial, we analyzed several real-world HWP malware techniques using **PE Explorer+**:

- Macro-based payload execution
- EPS/PostScript exploitation
- Embedded OLE objects
- HWP application exploits

As these examples demonstrate, PE Explorer+ makes it possible to quickly navigate suspicious HWP structures, inspect decompressed data, identify embedded objects, and trace malicious content down to the final payload.

Hopefully, these features will make HWP malware analysis a little easier and more efficient. ;)
