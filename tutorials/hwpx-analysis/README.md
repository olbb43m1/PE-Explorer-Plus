# HWPX Malware Analysis with PE Explorer+

HWPX is an open-standard document format used in South Korea, based on the **Open Word Processor Markup Language (OWPML)** specification. Similar to Microsoft's OOXML (Office Open XML), an HWPX document is essentially a ZIP archive containing document components structured as XML files.

In this tutorial, we will analyze three **in-the-wild (ITW)** malicious HWPX samples and explore PE Explorer+'s support for decrypting password-protected documents.

* (ITW) Single OLE
* (ITW) Multiple OLEs
* (ITW) Exploit
* Encrypted Document

---

## Case 1: Single OLE

![Case 1-1](images/Case-1/fig-1.png)

Let's start by opening the first sample.

The **Suspicious** section identifies `ole2.OLE`. Let's examine how this object is referenced within the document body.

![Case 1-2](images/Case-1/fig-2.png)

`section0.xml` contains the main document content.

Using the keyword search feature, we can locate the embedded OLE object referenced within the document. In this sample, the embedded OLE object is executed when the user clicks the corresponding object in the document.

![Case 1-3](images/Case-1/fig-3.png)

Now let's examine the OLE object itself.

Clicking **Open as new tab** allows us to load and inspect the OLE object directly.

![Case 1-4](images/Case-1/fig-4.png)

Selecting the `Ole10Native` stream reveals the embedded filename as well as the original path from which the object was inserted.

Now click **Open as new tab** to load `HancomReader.scr`, which is embedded inside the OLE object.

![Case 1-5](images/Case-1/fig-5.png)

We can easily extract and inspect the 32 KB malicious executable.

---

## Case 2: Multiple OLEs

![Case 2-1](images/Case-2/fig-1.png)

Opening the second sample reveals a total of seven OLE objects.

Let's first examine how these objects are referenced within the document body.

![Case 2-2](images/Case-2/fig-2.png)

Multiple OLE objects are referenced throughout the document. Let's take a closer look at one of them.

The hyperlink is presented to the user as a PDF document (`자료집-1.pdf`), while internally it points to an executable (`hancom-1.exe`).

Now let's inspect each of the seven embedded OLE objects.

![Case 2-3](images/Case-2/fig-3.png)
![Case 2-4](images/Case-2/fig-4.png)
![Case 2-5](images/Case-2/fig-5.png)
![Case 2-6](images/Case-2/fig-6.png)
![Case 2-7](images/Case-2/fig-7.png)
![Case 2-8](images/Case-2/fig-8.png)
![Case 2-9](images/Case-2/fig-9.png)

The seven OLE objects consist of three decoy documents, three malicious executables, and one malicious batch file:

* `ol1e.ole` : `Seminar-1.pdf` (PDF / Decoy)
* `ole2.ole` : `Seminar-2.pdf` (PDF / Decoy)
* `ol3e.ole` : `Seminar-3.pdf` (PDF / Decoy)
* `ole4.ole` : `hancom-1.exe` (EXE) ← Executable referenced in the example above
* `ole5.ole` : `hancom-2.exe` (EXE)
* `ole6.ole` : `hwp.exe` (EXE)
* `ole7.ole` : `hancom_service.bat` (BAT)

Now let's take a closer look at `hancom-1.exe`.

![Case 2-10](images/Case-2/fig-10.png)

The file paths extracted from the executable's strings show references to both the decoy document (`Seminar-1.pdf`) and the batch file (`hancom_service.bat`).

Based on these references, we can infer that when the user clicks the hyperlink in the document, the malicious executable (e.g., `hancom-1.exe`) is launched. It then opens the decoy document (`Seminar-1.pdf`) while also executing the batch file (`hancom_service.bat`).

The batch file processes an internally encoded script through multiple stages. A detailed analysis of this script is outside the scope of this tutorial.

---

## Case 3: Exploit

![Case 3-1](images/Case-3/fig-1.png)

Next, let's examine a malicious document that exploits **CVE-2015-6585**.

The document contains a single OLE object, `ole1.ole`. Let's first examine how it is referenced within the document.

![Case 3-2](images/Case-3/fig-2.png)

In the first document body file, `section0.xml`, we can identify where the `ole1.ole` object is embedded.

Unlike the previous samples, however, this object is not triggered by enticing the user to click it. Instead, the data within the object is referenced as part of the exploitation process.

![Case 3-3](images/Case-3/fig-3.png)

The root cause of the vulnerability can be found in `section1.xml`.

Scrolling through the XML reveals an unusual sequence that stands out from the surrounding data. In hexadecimal, the sequence is:

`E1 80 80 E1 88 9C`

The vulnerability is caused by a **type confusion** issue, and these six bytes are ultimately interpreted as the pointer value `0x121C1000`.

A detailed analysis of the vulnerability itself is outside the scope of this tutorial. For additional technical details, refer to [this analysis](https://media.kasperskycontenthub.com/wp-content/uploads/sites/43/2016/02/20081603/FireEye_HWP_ZeroDay.pdf).

* `E1 80 80` (UTF-8) → `1000` (UTF-16)
* `E1 88 9C` (UTF-8) → `121C` (UTF-16)

Now let's examine the OLE object referenced during exploitation.

![Case 3-4](images/Case-3/fig-4.png)

The object data is approximately **173 MB**, which is unusually large for an object embedded in a document.

Click **Open as new tab** to load and inspect the object.

![Case 3-5](images/Case-3/fig-5.png)

The object data has a **heap-spray structure** consisting of repeated `NOP + Shellcode` chunks.

The shellcode searches for an encoded payload using an embedded marker and then loads the payload. A detailed analysis of the shellcode and payload-loading process is outside the scope of this tutorial.

---

## Case 4: Encrypted Document

Finally, let's examine a document encrypted with a user-supplied password.

![Case 4-1](images/Case-4/fig-1.png)

When the document is opened, the **Summary** tab indicates that the document is encrypted and provides a **Decrypt** button.

![Case 4-2](images/Case-4/fig-2.png)

Examining the document body file, `section0.xml`, confirms that its contents are encrypted.

![Case 4-3](images/Case-4/fig-3.png)

Clicking the **Decrypt** button in the Summary tab opens the document decryption interface.

PE Explorer+ provides three methods for recovering or supplying the password:

* **Input Password**: If the password is already known, it can be entered directly.
* **Brute Force**: Attempts to recover the password using combinations of numbers, letters, special characters, and other configurable character sets.
* **Dictionary File**: Attempts to recover the password using a text-based password dictionary.

![Case 4-4](images/Case-4/fig-4.png)

The **Brute Force** mode provides several features for efficient password recovery.

It uses multithreading based on the available CPU cores to maximize performance. If part of the password is already known, the **Prefix/Postfix** options can significantly reduce the search space and make the cracking process more efficient.

![Case 4-5](images/Case-4/fig-5.png)

Once the password is successfully recovered, it is displayed at the bottom of the window and the document is automatically decrypted.

![Case 4-6](images/Case-4/fig-6.png)

Opening `section0.xml` again now reveals the decrypted document content.

> **Note:** The decryption feature also supports HWP documents.

---

## Conclusion

In this tutorial, we analyzed three **real-world malicious HWPX samples** and an encrypted HWPX document using PE Explorer+.

PE Explorer+ allows analysts to quickly identify suspicious elements by providing convenient browsing and keyword-search capabilities for XML-based document bodies and header files, along with integrated OLE object analysis.

For encrypted malicious documents, PE Explorer+ also provides **brute-force** and **dictionary-based password recovery**, allowing analysts to decrypt protected documents and continue their analysis.

> **Note:** Although not covered in this tutorial, **Distribution Documents**, which restrict document editing and printing, also contain internally encrypted data. As with the HWP analysis feature, PE Explorer+ automatically decrypts HWPX Distribution Documents for analysis.

As a newer document standard, HWPX is rapidly replacing the legacy OLE-based HWP format. As its adoption continues to grow, we can expect to encounter a wider variety of malware and attack techniques targeting HWPX documents.

I hope **PE Explorer+** proves useful in analyzing these threats and responding to document-based malware. ;)
