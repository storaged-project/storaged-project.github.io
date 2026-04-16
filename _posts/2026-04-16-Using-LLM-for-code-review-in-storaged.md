---
layout: post
title:  "Using LLM for Code Review in storaged Projects"
date:   2026-04-16 8:13:00 +0100
categories: other
author: Vojtech Trefny
---

For a couple of months now, we have been experimenting with using AI/LLM tools for code review for our projects under the [storaged.org](https://storaged.org) umbrella. We originally started with testing various tools for pull request reviews available on GitHub. For that, we are currently using [CodeRabbit](https://www.coderabbit.ai/).

But the newly added code is not the only code that should be reviewed. Our projects have a long history: the first version of libblockdev was released in 2014 and the history of UDisks goes back to 2008. The code, of course, went through many changes and rewrites since then and many people reviewed it. And we have tests and use static analysis and other tools to ensure code quality. But that doesn't mean there are no bugs or other issues. We know there are, our users report them somewhat regularly.

Libblockdev together with UDisks contain about 100 thousand lines of C code. Doing a full review of all that code would be a huge undertaking. For a human. A machine can do the review in a matter of minutes.

This is a good place to stop and address some questions and concerns you might have. Yes, LLMs, or what most people call AI these days, are not perfect, they make mistakes and results can be unpredictable or completely hallucinated. These are valid issues, but for this specific use case -- reviewing existing code written by humans -- these are less of a concern. Each issue found was first reviewed by the developer to make sure it is actually a real issue and the proposed fix was also reviewed before applying. And the resulting change went through the standard process before including it in the project -- pull request with a review by another developer.

We are also well aware that the code having been reviewed by these tools doesn't mean there aren't any issues anymore. But the issues that were found and fixed were real issues and fixing even one issue in any software is always a net positive regardless of whether the issue was found by tests, static analysis, a quality engineer or in this case an LLM/AI tool.

With this out of the way, let's move to the actual review process, and more importantly, the findings.

### The review

The review process itself is pretty simple. We are currently using [Anthropic's Claude](https://www.anthropic.com/claude) with the Opus 4.6 model. We did experiment with other tools and models as well, but so far Opus 4.6 seems to produce the best results (as of March 2026).

We used some basic phrases as prompts, without asking for any specific areas to review. A basic variant of "do code review of the existing code base" was enough for our use case (more about prompts and some unexpected results later). The produced review is usually a formatted numbered list of issues sorted by priority and well documented, with arguments supporting the findings. The short description of the issue was usually good enough for deciding whether the issue is real or a false positive.

> Crypto: Wrong strerror_l usage for ioctl errors in OPAL
>
> File: `src/plugins/crypto.c:3911`
>
> `ioctl` returns `-1` and sets `errno`, but the code uses `strerror_l(-ret, ...)` which gives `strerror_l(1, ...)` (always `EPERM`).
>
> Fix: Use `strerror_l(errno, c_locale)`.

<p style="text-align:center; font-style: italic;">Example of an issue found in libblockdev</p>

One thing we noticed quite quickly was that one review is not enough. The model seems to stop after a certain number of issues and after fixing these, running the prompt again produces a new set of issues. We did a number of runs for each project (both with and without a new context) and by the last run, the number of real issues found was close to zero. With a decreasing number of "real" issues in the code, the false-positives and "nitpicky" issues in the reports seemed to outnumber the newly discovered issues. This is understandable and kind of expected behavior and we've definitely seen human reviewers behave similarly when reviewing code that didn't have any glaring issues to point out.

We also experimented with limiting the review scope in the prompt: either to one specific module or plugin, or even to just one file. This approach seems to produce slightly better results within the limited scope, but also produces more false-positive reports as the model doesn't typically recognize relationships and implications at a global scale. But even with this caveat, this is still very well worth the effort.

### Findings

Now, let's talk about the issues that were found and fixed. In total, in all of the "review runs", 235 issues were reported: 110 in libblockdev and 125 in UDisks. (We didn't stop at these projects, our other projects went through the same process, but for the sake of simplicity, we'll focus only on these two here.)

Out of these, the largest group, 41 in total, were related to improper resource handling, mostly memory leaks, closely followed by various error handling issues. And because none of the developers working on these projects are native speakers, third place belongs to fixing typos, grammar and documentation with 27 issues fixed.

The "winners" here show where most of the existing issues in our code are hidden: in various error paths (upon closer inspection, most of the memory leaks were located in the error paths as well). This makes sense. These are hard to cover in automated tests and finding issues in error paths is mostly relying on manual testing, static analysis and of course, on code review.

Fixing grammar and typos might not seem that important, but these also include public API documentation where a misunderstanding about usage or memory ownership can lead to bugs in other projects using this API. And we even found a few nonsensical error messages that would definitely confuse users when shown.

> Crypto: Stale `strerror_l(0)` produces "Success" in error message
>
> **File:** `src/plugins/crypto.c:2255`
>
> `ret` is 0 (from successful `crypt_load`), so the error message reads
> "Label can be set only on LUKS 2 devices: Success".
>
> **Fix:** Remove the `strerror_l` call from this error message (it is a type-check
> error, not a syscall error).

<p style="text-align:center; font-style: italic;">Again in libblockdev. This very much resembles the famous "Task failed successfully" error message.</p>

### And what about CVEs?

Memory management, error paths and grammar. Important issues and it's good these are being fixed, but what about something more serious?

During our experiments with AI review, we didn't find any serious flaws or security issues. That, of course, doesn't mean there are none in our code, but if we knew about issues like these, we wouldn't be doing this exercise. What we can do is test the tools on a security issue we know exists. Or to be more precise, a security issue that existed.

In February, two related security issues were reported against UDisks -- [CVE-2026-26103](https://access.redhat.com/security/cve/cve-2026-26103) and [CVE-2026-26104](https://access.redhat.com/security/cve/cve-2026-26104). Both were caused by missing authorization checks in some UDisks public functions. UDisks is a daemon that runs with root privileges and allows unprivileged users to perform certain operations when some conditions (based on the operation, type of device, type of session the user is logged in, etc.) are met. We use Polkit for this. And in these two cases the authorization check was missing in the code. This went through review by two developers, and neither of us noticed. Now the question is, would AI notice?

Short answer is: yes, it would:

>  **Missing PolicyKit Authorization on LUKS Header Backup/Restore**
>
>  Files:
>  - `src/udiskslinuxencrypted.c:1356-1442` (`handle_header_backup`)
>  - `src/udiskslinuxblock.c:4238-4312` (`handle_restore_encrypted_header`)
>
>  Both D-Bus method handlers perform privileged operations without any PolicyKit authorization check. Compare with other handlers in the same file (lines 524, 784, 974, 1133, 1288) which all call
>  udisks_daemon_util_check_authorization_sync(). There are also no PolicyKit action IDs defined in data/org.freedesktop.UDisks2.policy.in for these operations.
>
>  Severity: CRITICAL

<p style="text-align:center; font-style: italic;">Both CVEs clearly pointed out in the review</p>

But even if this result looks impressive, it was not that easy. As I already mentioned, running the reviews multiple times and with slightly different prompts can change the results a lot. And the same happened here. The first two general "do code review" prompts did not uncover these issues. For the third run (with a new context), I explicitly asked to focus on potential security issues, and the tool happily obliged:

> I'll conduct a security-focused code review of the UDisks codebase. Since this is a privileged system daemon, security issues are critical. Let me explore several attack surface areas in parallel.

and one of the spawned agents explicitly did "review privilege and auth issues" in order to finally find this.

So even AI isn't infallible and doesn't just spot everything every time. On the other hand, neither did two senior software engineers working on the project when reviewing the pull request introducing this, so maybe we shouldn't start throwing stones either.

### What is "code review" and what exactly is "code"?

As mentioned multiple times already, the exact prompt wording matters. We have found that what we simply refer to as "code review" can lead to different results based on the wording and context.

Asking simply to "Do a code review on this project" usually leads to a high-level description of the project with a few key findings. These might include some specific issues and bugs, but more often the result is just a summary of overall "code health" of the project. In our case, praising our projects for clean and readable code and good test coverage. This is nice to read, but doesn't help much.

> ### Overview
> UDisks is a well-maintained ~160-file C project (v2.12.0) implementing a root-running storage management daemon using GLib/GObject and D-Bus. The codebase is in active maintenance with recent work focused on stability and safety hardening rather than new features.
>
> ...
>
> ### Security (Strong)
>
> For a daemon running as root with direct hardware access, the security posture is excellent:
>
> ...
>
> ### Code Quality Observations
>
> **Positives:**
> - Consistent coding style (GNU C, spaces-only, proper emacs modelines)
> - Naming conventions (`UDisks` prefix, `snake_case` functions) applied uniformly
>
> ...
>
> **Technical Debt:**
> - ~80+ `TODO`/`FIXME`/`XXX` comments scattered across the codebase. Notable clusters:
>
> ...
>
> ### Summary
> This is a mature, security-conscious codebase. The main areas for improvement are clearing the TODO backlog and tightening a few intermediate NULL checks in getter chains. No critical issues found.

<p style="text-align:center; font-style: italic;">Code review of UDisks. Some TODOs and FIXMEs, but other than that it's perfect, right?</p>

But if we ask to "Do a code review of the existing code base and report any issues or bugs you find" instead, it actually goes through the code more thoroughly and reports individual issues. And suddenly, our great project now has 16 issues, three of them high priority.

> ### Summary
>
> 16 issues found across the daemon core, modules, client library, and CLI tool.
>
> | Severity | Count | Key areas |
> |----------|-------|-----------|
> | High     | 3     | NULL derefs in partition ops (daemon crash), use-after-free in LVM2, double-close in LSM |
> | Medium   | 9     | Dead code in flag checks, empty DM name, insecure passphrase handling, memory leaks, wrong D-Bus completion |
> | Low      | 4     | Missing bounds checks, minor memory leaks in CLI tool, TOCTTOU race |

<p style="text-align:center; font-style: italic;">16 new issues found. Not that perfect after all.</p>

Another interesting thing we found is that "code" in the code review often doesn't include tests. This might be a specific problem for our projects, where tests are somewhat separated: for both libblockdev and UDisks the test suite is part of the same repository, but it is written in Python (in contrast to C) and for UDisks the biggest part of the test suite uses the DBus API, making it even more distinct. But that doesn't change the fact that without explicitly asking for "code review of the test suite", the tests were completely ignored. If included, issues in tests are actually the second biggest part of the issues found in this experiment. Looks like we might need tests for our tests. And that's only for libblockdev, UDisks "test suite code review" is still in progress.

And the same "tests are not code" issue exists for the other non-code parts of the project: documentation, man pages and various YAML config files. This shows that a simple one line prompt is not enough. For a one time review of the entire project as we did here, writing the prompts manually one by one and trying different tactics and approaches might be a good enough solution, but in the future more detailed instructions will surely be needed.

The obvious next step is to prepare a [skill for Claude](https://code.claude.com/docs/en/skills) that would include all these instructions for future code reviews. We don't intend to do a thorough code review of the entire existing code base for every change (that's where the AI-assisted code review on pull requests takes over), but we definitely intend to continue with this in the future and are looking forward to new and better models.

### Conclusion

Even though this started just as an experiment with the tools that are currently available for us, using Claude and Opus 4.6 for code review showed some really interesting results. We were able to quickly fix more than two hundred issues in our code (and that's counting only libblockdev and UDisks) and even though these were not critical or security bugs, this will definitely improve the project. We will continue working with AI/LLM tools and hopefully eliminate even more issues and also speed up bringing new features -- both by directly implementing these with the help of Claude and simply by offloading some of the other work to it.
