---
title: Tables turn, So does my project
date: 2025-08-02
author: Mankirat Singh
description: |
    Sharing my progress on how I am working on ABI compliance reporting for PostgreSQL as a BuildFarm Module. In this post, I walk through the new workflow, highlighting lessons learned from community feedback, and discuss upcoming improvements. If you're curious about ABI checking for Postgres Binaries and how it integrates with BuildFarm, this post is for you!
tags: ["PostgreSQL", "abicc"]
cover:
    image: "/posts/abi-compliance-reporting-v2/cover.jpg"
    alt: "banner image"
    relative: false
---

So, you might've caught my previous two blog posts where I explained on implementing an ABI compliance reporting system for PostgreSQL as a BuildFarm module. The initial approach involved cloning the PostgreSQL source code and looping through commits to pinpoint the exact commit responsible for any ABI breakage.

But, after discussing with the community on the hackers list([here](https://www.postgresql.org/message-id/flat/B1353FC2-4D33-4DC0-AB66-6CA037AD0AE9%40justatheory.com)), a few key issues came to light:

1.  The original approach didn't quite align with how the other BuildFarm modules functioned.
2.  Looping through commits introduced potential complications, especially when dealing with commits where source compilation might fail. Handling these edge cases would've added unnecessary complexity.
3.  Identifying the commit causing ABI breakage is usually pretty straightforward. Since the module typically runs on one or more BuildFarm animals, the number of commits between runs is small enough to easily spot the culprit.

These points led to a shift in how the module actually works. Let's call it **ABI Compliance Reporting V2**!

### ABI Compliance Reporting V2

BuildFarm animals usually run on cron jobs, triggering the `run_build.pl` script at set intervals. This script compiles and tests the source code for the latest commit. Interestingly, the BuildFarm client script clones the PostgreSQL source for a specific branch into the `$branch_root/pgsql` directory. Then, it copies the source (excluding the `.git` directory) to a separate location. This prevents the configure script from interfering with Git operations when generating configuration files.

So, by leveraging the `$branch_root/pgsql` Git directory, I can grab the latest tag for that branch using `git -C ./pgsql describe --tags --abbrev=0`.

Here's the new workflow:

1.  **Store the latest tag:** The module stores the latest tag in the `$abi_compare_root/$pgbranch/latest_tag` file.
2.  **Checkout the tag:** Check out the stored tag.
3.  **Copy the source:** Copy the source code (without the `.git` directory).
4.  **Compile:** Compile the source code.
5.  **Generate ABI XMLs:** Store the generated ABI XMLs in `$abi_compare_root/$pgbranch/$latest_tag/xmls/`.

For subsequent runs, the module checks if the `latest_tag` file exists. If it does, it assumes the corresponding XMLs are also present. It then directly uses the binaries from the latest commit and compares their XMLs with the most recent tag's XMLs to generate a report.

Currently, the `abicheck` directory (or `$abi_compare_root`, which houses all run-related files for this module) looks like this:

```
abicheck.lucario/
└── HEAD
    ├── diffs
    │   ├── libpq.so-REL_18_BETA1-HEAD.log
    │   └── postgres-REL_18_BETA1-HEAD.log
    ├── latest_tag
    ├── logs
    ├── REL_18_BETA1
    │   ├── build_logs
    │   │   ├── build.log
    │   │   ├── configure.log
    │   │   └── install.log
    │   ├── pgsql/...
    │   ├── inst/...
    │   └── xmls
    │       ├── ecpg.abi
    │       ├── libpq.so.abi
    │       └── postgres.abi
    └── xmls
        ├── ecpg.abi
        ├── libpq.so.abi
        └── postgres.abi
```

*   `HEAD` represents the current branch.
*   `REL_18_BETA1` is the latest tag.
*   The `xmls` directory contains the ABI XMLs generated for the current commit.
*   The `diffs` directory holds the ABI diffs between the latest commit build and the latest tag.
*   Inside the `REL_18_BETA1` directory, the `xmls` directory contains the ABI XMLs generated for the latest tag.

I've also added support for `meson` and `msvc` build systems!(Yes, I copied the code from the run_build.pl script :P)

If you want to check the latest source for the module, you can find it on this [GitHub pull request](https://github.com/PGBuildFarm/client-code/pull/38) and to test it you may use the following command.
```bash
perl ./run_build.pl --nosend --verbose --only-steps="configure make make-doc install abi_comp-check" --nostatus --keepall
```
In the past weeks, I've also been trying to remove the Git CLI dependency from my module, as other modules also use it. However, it seems BuildFarm renames the branch it's working with to `bf_$branch_name`. Apparently, The `bf_` prefix dates back to the dark days of history when Postgres used CVS and when git was adopted, there was a switchover period when both could be used, so the same naming scheme is still being used :) - according to the BuildFarm maintainers.

### What's on the horizon?

*   If a commit is itself a tag, store its XMLs permanently.
*   Remove unnecessary files, respecting the `--keepall` option.
*   Disallow execution if Git isn't being used.
*   Inquire about allowing the head branch.
*   Compare from the base commit of the branch if no latest tag is found.


### Thanks for reading!