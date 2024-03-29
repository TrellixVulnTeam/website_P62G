---
breadcrumbs:
- - /chromium-os
  - Chromium OS
- - /chromium-os/packages
  - packages
- - /chromium-os/packages/crash-reporting
  - Crash Reporting (Chrome OS System)
page_name: faq
title: Crash Reporting FAQ
---

[TOC]

---

## General Questions

### How can I get the stack trace from a crash on my development Chromebook?

Crash reporter will still collect crashes, they just won't be sent to the Crash
Server. Beyond that, the crash reporter is no longer involved, but you can get
more info from [Getting a stack dump from a minidump
file](https://sites.google.com/a/google.com/chromeos/resources/engineering/getting-a-stack-dump-from-a-minidump-file).

### How can I get a core dump from a minidump for use by gdb?

There's a `minidump-2-core` executable provided by Breakpad to convert a
minidump to a core file. You can build it in a Chrome checkout with
`ninja -C out/Default minidump-2-core`, or you can build it in a Chrome OS
checkout with `sudo emerge google-breakpad`.

Once you have `minidump-2-core`, you can look at section [Use gdb to show a
backtrace](http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/crash-reporting/debugging-a-minidump#TOC-Use-gdb-to-show-a-backtrace)
of the [Debugging a Minidump
File](/chromium-os/packages/crash-reporting/debugging-a-minidump) guide. This is
my collection of instructions for how I've made use of `gdb` given a minidump
file. It shows, for example, how to adjust the symbol addresses in `gdb` so they
match up appropriately, and it includes instructions for how to do this for ARM
minidumps as well.

Googlers: You can also look through the section [Crash dump
analysis](https://sites.google.com/a/google.com/chrome-msk/dev#TOC-Crash-dump-analysis),
which is where I found the address-adjusting logic.

### Is there a design document for Chrome OS Crash Reporting?

Yes, in the [project
sources](https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/crash-reporter/docs/design.md).
For a general overview, though:

For **non-Chrome** processes we use an external program, `crash_reporter`, that
gets called by the kernel when a crash occurs. It gets passed the crashing
process's core file, and takes care of generating the report. `crash_reporter`
also generates a report for kernel crashes, unclean shutdowns, and several other
things.

For **Chrome** processes, it uses Breakpad. When a Chrome crash occurs, the
Breakpad library code linked into it takes care of generating a report. Then it
calls crash_reporter to queue the full report.

The crash reports generated by `crash_reporter` are uploaded by `crash_sender`,
which is run via a cron job every hour.

We also report crash metrics (UMA) for Chrome OS. The crash metrics for
**Chrome** are reported and uploaded by Chrome itself. The crash metrics for
**non-Chrome** processes are reported by `crash_reporter` *via* Chrome, which
then uploads them. `crash_reporter` sends a metric for each crash, saying
whether it's a user crash, kernel crash, or unclean shutdown.

There are currently two different, but redundant, types of metrics being
reported by `crash_reporter`:

*   The old type uses the "Logging.CrashCounter" histogram with buckets
            for "user", "kernel", and "unclean shutdown" counts.
*   The new type has Chrome report them in its own stability metrics
            (see <https://crbug.com/193643> for more info).

### What's the difference between "Aw, Snap!" and "He's Dead, Jim!"?

These are error pages shown by Chrome when a tab's process in some way dies.
According to the Chrome Help pages, "You may see the "Aw, Snap!" message if a
webpage's process crashes unexpectedly." On the other hand, "You may see the
“He’s Dead, Jim!” message if the operating system has terminated the tab’s
process due to a lack of memory. Alternatively, if you terminated the process
using Google Chrome's Task Manager, the system's task manager, or with a command
line tool, this message will appear as well."

That is, an "Aw, Snap" is most likely caused by a genuine crash of the
non-browser process (e.g. renderer, plugin), and should result in a crash report
if consent is enabled (see FAQ entry [How can I know if my Chromebook will
report
crashes](https://sites.google.com/a/google.com/chromeos/resources/engineering/crash-reporting-faq#TOC-How-can-I-know-if-my-Chromebook-will-report-crashes-)).
The "He's Dead, Jim" is shown if a webpage's process is terminated by somebody
or something else.

Reference ("Aw, Snap!"): <https://support.google.com/chrome/answer/95669>
Reference ("He's Dead, Jim!"):
<http://support.google.com/chrome/bin/answer.py?hl=en&answer=1270364>
Reference: <https://crbug.com/219693>

---

## Crash Reporter

### Will a developer's build image upload crash reports?

No. A crash will still be processed by `crash_reporter`, but the report will not
be uploaded to the Crash Server. More specifically, `crash_sender` will only
send crash reports if the word "Official" appears in the
"CHROMEOS_RELEASE_DESCRIPTION" line of `/etc/lsb-release`.

Googlers: You can override this behavior by running `crash_sender` manually with
the `--dev` option. This may be helpful for certain testing scenarios. By
default, `crash_sender` will delay up to ten minutes before sending each crash
report, so you will probably also want to set the command line option
`--max_spread_time` to 0 to make them upload right away. Recipe for testing
crashes (as root, on a test image):

```none
sleep 100 &
kill -SEGV $!   # generate a core
metrics_client -C   # force metrics consent
/sbin/crash_sender --dev --max_spread_time=0   # force the upload
grep crash_sender /var/log/messages
```

### How can I know if my Chromebook will report crashes?

You can check the consent settings. Go to **Settings -&gt; Advanced Settings**
and then see if "Automatically send usage statistics and crash reports to
Google" is enabled under the **Privacy** section. If there is no such setting
shown, then you are running a developer's build image (see FAQ entry [Will a
developer's build image upload crash
reports](#TOC-Will-a-developer-s-build-image-upload-crash-reports-)).

### Why aren't crashes being reported for Chrome?

First, check to make sure that consent has been enabled (see FAQ entry [How can
I know if my Chromebook will report
crashes](#TOC-How-can-I-know-if-my-Chromebook-will-report-crashes-)). If no such
setting is shown, read on.

If you don't see any reference at all to crash reporting in
`/var/log/ui/ui.LATEST` after a crash, chances are you're running a developer
image. Developer images are built with Chromium. Chrome handles its own crashes
by linking in Breakpad, whereas Chromium does not link in Breakpad. Therefore,
when a Chromium crash occurs, it is simply ignored.

If you want to see these crash reports, you can either have crash_reporter
report Chromium crashes, or build Chrome instead of Chromium. For the former,
see instructions related to `collect_chrome_crashes`, below. For the latter, use
the "--internal" flag to cros chrome-sdk, as explained in the instructions for
[building Chrome on Chrome
OS](/chromium-os/how-tos-and-troubleshooting/building-chromium-browser).

Bear in mind that crash_reporter won't upload crash reports by default for
developer images. See FAQ entry [Will a developer's build image upload crash
reports](#TOC-Will-a-developer-s-build-image-upload-crash-reports-) for more
information.

### Are there limits on how many crashes will be reported?

Yes.

`crash_reporter` will stop creating crash reports if there are 32 of them
already on disk. There's a separate limit per collection directory, so
`/var/spool/crash` and `/home/chronos/user/crash` can each contain up to 32
crash reports. That limit is defined in **crash-reporter/crash_collector.cc**'s
variable **CrashCollector::kMaxCrashDirectorySize**. With respect to uploading
crash reports, `crash_sender` has a per day limit. More specifically, if at
least 32 crash reports have been sent within the past 24 hours, and the combined
compressed size of those reports is above 24 MB, it will delay and try sending
the crash report later. These limits are defined by **kMaxCrashRate**
andkMaxCrashBytes** in **crash-reporter/crash_sender_util.h**.

### Where can I find the crash minidump/core file for a crashed process on my Chromebook?

#### Official builds:

For processes run as user "chronos", `crash_reporter` puts its files (\*.dmp and
\*.meta) in `/home/chronos/user/crash/` if the user is logged in,
`/home/chronos/crash` if not. For other processes, `crash_reporter` puts them in
`/var/spool/crash/`.

#### Developer builds:

For processes run as user "chronos", `crash_reporter` puts its files (\*.dmp,
\*.meta, and \*.core) in `/home/chronos/crash/`. For other processes,
`crash_reporter` puts them in `/var/spool/crash/`. If you are running a
developer image, the `/root/.leave_core` file should exist, so `crash_reporter`
will not delete the core file.

### Where can I find the crash minidump/core file when chrome crashes immediately after login and logs me out?

For official builds, minidumps are written to \`/home/user/\*/crash\`
directories, which are only mounted when the corresponding user(s) are logged
in. cryptohome unmounts the home directory if chrome crashes immediately after
login and leaves crash dump. <http://crbug.com/857317>. You can use the
cryptohome command to mount and decrypt home directories:

```none
DUT$ cryptohome --action=mount_ex --user=user@gmail.com
DUT$ ls -l /home/user/*/crash/
```

### Why are my crash dumps disappearing sometimes?

The `crash_sender` process runs every hour. In most cases when it runs, it will
delete your existing crash dumps -- after uploading the report if that is
enabled. The exception to them being deleted is if `crash_sender` is uploading
crash reports and it reaches a limit, in which case it will hold off for another
hour. You can prevent the `crash_sender` from running by touch'ing the
`/var/lib/crash_sender_paused` file.

### At what point can the crash reporter catch crashes?

TBD

### Something crashed during startup, but I don't see it in /var/spool/crash/ or ~/crash/?

In order for the crash reporter to be called to process a crash, the line in
`/proc/sys/kernel/core_pattern` must start with "|/sbin/crash_reporter". Unless
otherwise set, it defaults to just "core". The `crash_reporter` program sets the
kernel's core pattern when it is first run by Upstart. This is currently done at
the same time as "system-services" services (see
`src/platform/init/crash-reporter.conf`).

Reference: <https://crbug.com/199893>

### Do we report out-of-memory (OOM) crashes?

No. When a system is running out of memory, the kernel will invoke its OOM
killer to kill some process with the hopes that it will free up enough memory.
On Chrome OS the killed process should always be one of Chrome's, because we
make all others unkillable. Search for "init on ChromeOS" in [Out of memory
handling](http://www.chromium.org/chromium-os/chromiumos-design-docs/out-of-memory-handling)
to see which processes should be killed first.

When the OOM killer runs, you should see a message like the following in the
system's log:

```none
<4>[ 2461.625535] chrome invoked oom-killer: gfp_mask=0x200da, order=0, oom_adj=0, oom_score_adj=0
```

..followed by a bunch of current memory information and then:

```none
<3>[ 2461.638097] Out of memory: Kill process 9250 (chrome) score 648 or sacrifice child
<3>[ 2461.638107] Killed process 9250 (chrome) total-vm:197184kB, anon-rss:14464kB, file-rss:13184kB
```

Unfortunately, we don't have a good way to report these OOM killings. When the
kernel kills a process with the oom-killer, it effectively does so with the
SIGKILL signal. Because of this, the Breakpad signal handler does not get
invoked (which is how Chrome reports its crashes), nor does `crash_reporter` get
called by the kernel.

There's been talk about handling this within Chrome by having a soft memory
limit. See [Out of memory
handling](http://www.chromium.org/chromium-os/chromiumos-design-docs/out-of-memory-handling)
for the design doc. Otherwise, I imagine we could modify the kernel to do
something smarter than a SIGKILL. A simpler solution would be to have something
monitor the system logs, and simply report the OOM killing (at least with an UMA
metric).

---

## Core File Questions

### Does crash_reporter save the core file for a crash?

Yes, but only for developer images. When a crash occurs, the kernel sends the
core file to `crash_reporter`. The core file is saved to disk by
`crash_reporter` and then converted to a minidump. If the `/root/.leave_core`
file exists (i.e. it's a developer image), the core file will be left on disk.
See FAQ entry [Where can I find the crash dump/core file for a crashed process
on my
Chromebook](#TOC-Where-can-I-find-the-crash-minidump-core-file-for-a-crashed-process-on-my-Chromebook-)
for where to find the core file. Note that by default Chrome crashes are not
handled by `crash_reporter` (see FAQ entry [Why would chrome crashes not
generate a core file on dev
builds](#TOC-Why-would-Chrome-crashes-not-generate-a-core-file-on-dev-builds-)).

### How can I get the core file for a crash?

For non-Chrome crashes when running a developer image, see FAQ entry [Does
crash_reporter save the core file for a
crash](#TOC-Does-crash_reporter-save-the-core-file-for-a-crash-). For Chrome
crashes on a developer image, see FAQ entry [Why would chrome crashes not
generate a core file on dev
builds](#TOC-Why-would-Chrome-crashes-not-generate-a-core-file-on-dev-builds-).

If you're not running a developer image, the easiest way to get core files is
probably to touch the `/root/.leave_core` and
/mnt/stateful_partition/etc/collect_chrome_crashes files.

If you don't want to involve crash_reporter at all, for whatever reason, you can
manually change the setting such that the kernel creates a core file instead of
piping it to `crash_reporter`. First, set the core file pattern (make sure the
core's path is writable by the would-be crashing process):

> `sudo sh -c 'echo "/home/chronos/core.%e.%p" > /proc/sys/kernel/core_pattern'`

Then, modify the maximum size of core files created for the process(es) you care
about:

> `prlimit --core=unlimited --pid <pid>`

### Why would Chrome crashes not generate a core file on dev builds?

By default, all but Chrome crashes are handled by `crash_reporter`. Chrome
handles its own crashes by linking in Breakpad, which does not generate core
files. In order to have `crash_reporter` not ignore Chrome crashes, though, you
can touch the `/mnt/stateful_partition/etc/collect_chrome_crashes` file. This
file is normally used by the autotests in order for Chrome crashes to still be
handled. See also FAQ entry [Does crash_reporter save the core file for a
crash](#TOC-Does-crash_reporter-save-the-core-file-for-a-crash-).

Reference: <https://crbug.com/201089>

---

## Build Questions

### Should we be building with "-g", "-ggdb", or "-ggdb2"?

Use "-g". We used to build with "-ggdb" just to be more explicit about what
`cros_generate_breakpad_symbols` expects; however, WebKit currently relies on it
being "-g" if we want to remove its debug symbols. davidjames@ did the work to
figure out that "-g" and "-ggdb" are the same for the GNU compiler we're
currently using, so he could switch Chrome to building with "-g". To be
consistent, we might as well build everything with "-g". As of 11/10/2011 no
other packages had been modified to use the new option yet, but this is the
direction we'd like to go in.

Reference: <https://gerrit.chromium.org/gerrit/11462>

---

## Technical Details

### Can my program catch SIGSEGV without screwing up crash_reporter?

*Note: This discussion focuses on SIGSEGV, but it applies to all signals that
the kernel creates coredumps for (e.g. SIGQUIT, SIGILL, SIGABRT, etc...).*

Yes. Normally, if you don't catch SIGSEGV, the kernel will default to spawning
`crash_reporter` for the crash. If you catch SIGSEGV, then what happens depends
on how the segfault was sent and how you handle it. If the signal was sent to
your process by someone (e.g. using the `kill` command) then, after your signal
handler runs, your program will continue where it left off. If the segfault was
caused by something like an actual bad memory access (e.g. "\*(char \*)0x0 = 1")
then, after your signal handler runs, the signal could simply be sent again
(assuming your signal handler didn't change the runtime environment so as to
"fix" the source of the segfault). Chances are you don't want to just return
normally from your signal handler, though; in the second scenario you could
easily end up with an infinite loop.

If you want to **bypass** `crash_reporter`, you should be able to just call
`_exit()` from your handler (normally you want `_exit()` rather than `exit()` as
the latter will run `atexit()` hooks which could themselves could cause problems
in a signal handler context). In both scenarios that will bypass the kernel's
handling of the SIGSEGV. However, if you **want** `crash_reporter` to still run
after your handler finishes, you have to do two different things in order to
handle both scenarios. For the case where your program caused a segfault, you'll
want to set the SIGSEGV handler back to what it was before -- so that when the
signal is sent again it's handled as if you had no handler. In C this is done
with `signal(SIGSEGV, SIG_DFL)`. For the case where your program was explicitly
sent a signal by someone else, you'll have to re-send the signal to yourself.
You should be able to do this by just calling `kill()` directly within your
handler. Note that this should work without changing the SIGSEGV handler back
because the signal will still have been masked until your handler returns (i.e.
that's done in case your handler were to segfault). You can determine which
scenario you're in by checking the `si_pid` field of the `siginfo_t` struct your
handler is sent.

An example of how to catch SIGSEGV without screwing up `crash_reporter` can be
found in Google Breakpad's
[src/breakpad/src/client/linux/handler/exception_handler.cc](http://code.google.com/p/google-breakpad/source/browse/trunk/src/client/linux/handler/exception_handler.cc?spec=svn1020&r=1018#223):

```none
void ExceptionHandler::SignalHandler(int sig, siginfo_t* info, void* uc) {
  /* PUT YOUR HANDLER CODE HERE */
  if (info->si_pid) {
    // This signal was triggered by somebody sending us the signal with kill().
    // In order to retrigger it, we have to queue a new signal by calling
    // kill() ourselves.
    if (tgkill(getpid(), syscall(__NR_gettid), sig) < 0) {
      // If we failed to kill ourselves (e.g. because a sandbox disallows us
      // to do so), we instead resort to terminating our process. This will
      // result in an incorrect exit code.
      _exit(1);
    }
  } else {
    // This was a synchronous signal triggered by a hard fault (e.g. SIGSEGV).
    // No need to reissue the signal. It will automatically trigger again,
    // when we return from the signal handler.
  }
 
  // As soon as we return from the signal handler, our signal will become
  // unmasked. At that time, we will  get terminated with the same signal that
  // was triggered originally. This allows our parent to know that we crashed.
  // The default action for all the signals which we catch is Core, so
  // this is the end of us.
  signal(sig, SIG_DFL);
}
```

In fact, if you link in Breakpad you can just use its handler -- which will call
any callbacks you specify. This is how Chrome handles its crashes (see
[EnableCrashDumping() in
src/chrome/app/breakpad_linux.cc](http://code.google.com/searchframe#OAMlx_jo-ck/src/chrome/app/breakpad_linux.cc&type=cs&l=407)
for an example).

Reference:
<http://www.linuxquestions.org/questions/programming-9/sigsegv-handler-segmentation-fauld-handler-277790/>
Reference: <http://www.openqnx.com/phpbbforum/viewtopic.php?t=6835>
Reference:
<http://www.alexonlinux.com/how-to-handle-sigsegv-but-also-generate-core-dump>
Reference:
<http://www.justskins.com/forums/how-to-ignore-sigsegv-104217.html#post337119>

### For Chrome what general functions are used for reporting crashes?

Here is an ordered list of notable files & functions in the Chromium source tree
that are used by Chrome to report crashes on Chrome OS. Although most of this
probably applies to the Linux platform as well, I wrote these notes with Chrome
OS in mind.

**chrome/browser/chrome_browser_main_linux.cc: IsCrashReportingEnabled()**
Determines whether or not crash reporting should be done in Chrome.

#### *Chrome Browser Crashes*

**chrome/app/breakpad_linux.cc: EnableCrashDumping()**
Enables crash reporting for the browser. Determines the path for a browser
crash's minidump.

**breakpad/src/client/linux/handler/exception_handler.cc:
ExceptionHandler::HandleSignal()**
Handles the crash signals in the browser. Calls GenerateDump() to dump the
crash.

**breakpad/src/client/linux/handler/exception_handler.cc:
ExceptionHandler::GenerateDump()**
Creates a new process with clone() -- which calls Breakpad's WriteMinidump() to
do the dumping of the browser process.

**breakpad/src/client/linux/minidump_writer/minidump_writer.cc:
WriteMinidump()**
Attaches to and dumps the browser process to a minidump file.

**chrome/app/breakpad_linux.cc: HandleCrashDump()**
Reads the minidump file, adds additional MIME information, and either uploads it
or writes it out to file.

#### *Chrome Renderer Crashes*

**content/browser/child_process_launcher.cc: LaunchInternal()**
Seems to do the launching for renderers (e.g. sets kCrashDumpSignal).

**chrome/app/breakpad_linux.cc: EnableNonBrowserCrashDumping()**
Enables crash reporting for a renderer.

**breakpad/src/client/linux/handler/exception_handler.cc:
ExceptionHandler::HandleSignal()**
Handles the crash signals in a renderer. Calls the renderer crash handler; it
does not call GenerateDump() to do its own crash dumping.

**chrome/app/breakpad_linux.cc: NonBrowserCrashHandler()**
Called by a renderer process to handle its own crash. Writes to the browser’s
pipe with basic context info about the crash.

**chrome/browser/crash_handler_host_linux.cc:
CrashHandlerHostLinux::OnFileCanReadWithoutBlocking()**
Called by the browser process when a renderer crashes. Reads the basic crash
info from the renderer's pipe.

**chrome/browser/crash_handler_host_linux.cc:
CrashHandlerHostLinux::WriteDumpFile()**
Called by the browser process. Determines the path for a renderer crash's
minidump. Calls Breakpad's WriteMinidump() to do the dumping.

**breakpad/src/client/linux/minidump_writer/minidump_writer.cc:
WriteMinidump()**
Attaches to and dumps a renderer process to a minidump file.

**chrome/app/breakpad_linux.cc: HandleCrashDump()**
Reads the minidump file, adds additional MIME information, and either uploads it
or writes it out to file.

---

## About the Team

### Where should I file Crash Reporting bug reports/feature requests?

Use the chromium-os issue tracker: <https://crbug.com/new>. File it under
component "OS&gt;Systems&gt;CrashReporting".

### Who is responsible for this FAQ?

The Crash Reporting team. If you find any issues with the questions/answers, or
there's a question you feel would be beneficial to have answered here, feel free
to [create an issue](https://crbug.com/new) under area "Area-Logging" with type
"Type-Documentation".

### Who are the authors of this FAQ?

This FAQ is maintained by the Crash Reporting team. The original author was
Michael Krebs (mkrebs@). It started as part of a general Chrome OS reference
guide, but was later split into this FAQ, a [Crash Reporting Reference
Guide](https://sites.google.com/a/google.com/mkrebs/references/crash-reporting),
a [Google Breakpad Reference
Guide](https://sites.google.com/a/google.com/mkrebs/references/google-breakpad),
and another [Chrome OS Reference
Guide](https://sites.google.com/a/google.com/mkrebs/references/chrome-os).