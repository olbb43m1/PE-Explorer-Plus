# DOC Malware Analysis with PE Explorer+

Malicious Microsoft Word documents based on the Compound File Binary
(CFB) format commonly rely on VBA macros to execute malicious code.

In this tutorial, we walk through four in-the-wild DOC malware samples
and demonstrate how **PE Explorer+** can be used to efficiently perform
static analysis, with a particular focus on VBA macro extraction and
deobfuscation.

## Case 1

\<images/Case-1/fig-1.png\>

When the sample is opened, PE Explorer+ displays basic document
information along with suspicious elements detected in the file,
including VBA macros.

The **Warnings** section also indicates that the size of the `dir`
stream, which contains metadata required to interpret the VBA project,
has been manipulated. This type of malformed metadata is commonly used
as an anti-analysis technique to interfere with static analysis tools
and prevent them from correctly locating or extracting VBA macros.

Despite the manipulated stream metadata, PE Explorer+ successfully
identifies and extracts the macro streams.

Let's start by clicking:

`Macros → VBA → ThisDocument`

\<images/Case-1/fig-2.png\>

The **Default** tab shows the original VBA source code. As shown above,
a number of repetitive comment strings have been inserted
throughout the code to reduce readability and make manual analysis more
cumbersome.

Elements that are irrelevant to the actual program logic are removed in
the **Deobfuscated** view.

\<images/Case-1/fig-3.png\>

With the unnecessary comments removed, the resulting source code is significantly more readable, making the actual macro logic easier to inspect.

## Case 2

\<images/Case-2/fig-1.png\>

The second sample contains two VBA macro streams. The main execution
logic is located in `ThisDocument`.

\<images/Case-2/fig-2.png\>

The macro uses lightweight string obfuscation to hide parts of its
behavior.

Let's switch to the **Deobfuscated** tab.

\<images/Case-2/fig-3.png\>

Potentially suspicious constructs are highlighted at the top of the
analysis view, allowing analysts to quickly identify code that deserves
further investigation.

PE Explorer+ also performs basic static deobfuscation of string
expressions using techniques such as **constant folding** and **constant
propagation**.

As a result, many statically resolvable expressions are simplified
automatically, making the macro's behavior easier to understand without
manually reconstructing each string.

## Case 3

\<images/Case-3/fig-1.png\>

In this sample, we can immediately see that the document contains two
VBA macros and one embedded object.

Let's open:

`Macros → VBA → ThisDocument`

\<images/Case-3/fig-2.png\>

The macro constructs a string, stores the result in the `EYGASUID`
variable, and passes it to the `Mshduwh` function in `Module1`.

After resolving the string construction, the resulting value is:

`%TEMP%\8tr.exe`

This strongly suggests that the macro expects an executable payload at
that location.

Let's follow both the function reference and the embedded file.

\<images/Case-3/fig-3.png\>

The `Mshduwh` function in `Module1` is straightforward: it receives the
string as an argument and passes it to the VBA `Shell` function for
execution.

In other words, the macro ultimately attempts to execute:

`%TEMP%\8tr.exe`

\<images/Case-3/fig-4.png\>

The `8tr.exe` payload can be found inside the document's `ObjectPool`
storage.

PE Explorer+ allows the embedded file to be loaded directly through the
**Open as new tab** button, making it possible to continue the analysis
without manually extracting and reopening the payload.

\<images/Case-3/fig-5.png\>

Once loaded, PE Explorer+ provides basic static analysis information for
the embedded PE binary, including its **resources**, **strings**, and
other PE-related metadata.

This makes it possible to move directly from analyzing the malicious
document to inspecting its embedded executable payload.

## Case 4

\<images/Case-4/fig-1.png\>

The final sample contains four VBA macros, with the main execution logic
located in `ThisDocument`.

\<images/Case-4/fig-2.png\>

The macro references three properties from the `TadaSHC` module:

`TadaSHC.Label2.Tag`

`TadaSHC.Label1.Tag`

`TadaSHC.Tag`

Let's examine where these values come from.

The Label controls are stored in the `TadaSHC.f` stream.

\<images/Case-4/fig-3.png\>

The internal structure of the `f` stream is not particularly simple, but
even basic string extraction can reveal useful artifacts.

In this sample, the relevant Label values resolve to:

-   `Label1`: `"msiexec.exe"`
-   `Label2`: `"Wscript.Shell"`

These strings provide important clues about how the macro executes the
next stage.

\<images/Case-4/fig-4.png\>

The value of `TadaSHC.Tag` is stored in the `VBFrame` stream.

Inspecting this value reveals the remote location of the MSI package
used by the malware.

Putting the pieces together, the sample creates a `Wscript.Shell` object
and uses it to launch `msiexec.exe`. The command retrieves a remote MSI
package and executes it silently using the `/q` option.

This example demonstrates why analyzing only the visible VBA source is
not always sufficient. Important configuration data and strings may also
be stored in UserForm-related streams and referenced indirectly by the
macro.

------------------------------------------------------------------------

# Conclusion

In this tutorial, we analyzed four real-world malicious DOC samples
using PE Explorer+.

The examples primarily focused on VBA macro analysis and demonstrated
how PE Explorer+ combines document structure inspection, macro
extraction, suspicious-code highlighting, and lightweight static
deobfuscation to make malicious document analysis more efficient.

The deobfuscation engine uses static analysis techniques such as
constant folding and constant propagation where applicable. It does
**not** execute VBA code or rely on emulation to resolve obfuscated
expressions.

This approach has an important trade-off: it avoids executing
potentially malicious macro code during analysis, but it also means that
expressions requiring runtime state or more complex execution semantics
may not be fully resolved.

The current implementation is therefore intended to assist analysts
rather than replace manual analysis. Additional deobfuscation
capabilities may be introduced in future versions as needed.

We hope these features make PE Explorer+ useful for quickly triaging and
investigating malicious DOC files.
