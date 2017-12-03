# GDB安装（Mac OS X）

GDB在MacBook上安装后，需要完成下面的工作，才可以保证GDB正常工作：

- 给GDB授权。GDB必须对进程有完全的控制权，才可以对该进程进行Debug。这在Darwin系统上是默认禁止的。具体授权的方法是：<https://gcc.gnu.org/onlinedocs/gcc-4.8.1/gnat_ugn_unw/Codesigning-the-Debugger.html>

> ### I.1 Codesigning the Debugger
>
> The Darwin Kernel requires the debugger to have special permissions before it is allowed to control other processes. These permissions are granted by codesigning the GDB executable. Without these permissions, the debugger will report error messages such as:
>
> ```
>      Starting program: /x/y/foo
>      Unable to find Mach task port for process-id 28885: (os/kern) failure (0x5).
>       (please check gdb is codesigned - see taskgated(8))
>
> ```
>
> Codesigning requires a certificate. The following procedure explains how to create one:
>
> - Start the Keychain Access application (in /Applications/Utilities/Keychain Access.app)
> - Select the Keychain Access -> Certificate Assistant -> Create a Certificate... menu
> - Then:
>   - Choose a name for the new certificate (this procedure will use "gdb-cert" as an example)
>   - Set "Identity Type" to "Self Signed Root"
>   - Set "Certificate Type" to "Code Signing"
>   - Activate the "Let me override defaults" option
> - Click several times on "Continue" until the "Specify a Location For The Certificate" screen appears, then set "Keychain" to "System"
> - Click on "Continue" until the certificate is created
> - Finally, in the view, double-click on the new certificate, and set "When using this certificate" to "Always Trust"
> - Exit the Keychain Access application and restart the computer (this is unfortunately required)
>
> Once a certificate has been created, the debugger can be codesigned as follow. In a Terminal, run the following command...
>
> ```
>      codesign -f -s  "gdb-cert"  <gnat_install_prefix>/bin/gdb
>
> ```
>
> ... where "gdb-cert" should be replaced by the actual certificate name chosen above, and <gnat_install_prefix> should be replaced by the location where you installed GNAT.

- 对于Mac OS X 10.12以上版本，GDB还有些不太兼容，需要执行下面步骤。否则，gdb start无法成功。

> Create a .gdbinit file in your home-direcetory and write "set startup-with-shell off" in it.
>
> File can be created using `vi ~/.gdbinit`.
>
> Open a new terminal and gdb will work.

- 执行GDB时，使用sudo gdb执行，否则出现下面错误（与未授权的现象相同）。

> Unable to find Mach task port for process-id 678: (os/kern) failure (0x5).
>
>  (please check gdb is codesigned - see taskgated(8))

