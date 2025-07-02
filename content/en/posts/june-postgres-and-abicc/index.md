---
title: June, BuildFarm and ABIs
date: 2025-07-01
author: Mankirat Singh
description: This blog discusses the current progress and implementation of an automated ABI compliance reporting system for PostgreSQL, which is also my GSoC 2025 project.
tags: ["PostgreSQL", "abicc"]
---

If you might have read my previous blog posts, you would know that I am working on an automated ABI compliance reporting system for PostgreSQL as part of my GSoC 2025 project. In this blog post, I will discuss the current progress and implementation details of this system, as well as its integration with the PostgreSQL build farm.


## Current Progress
According to my estimated timeline, I should have completed the basic implementation of the ABI compliance checker module and started receiving reports on the server by now. With a slight delay, the client implementation is almost complete (working on some changes) and the server implementation is in progress (trying to get better with CGI scripts).

## Client Implementation
First things first - If you want to jump directly to the code, you can find it in this [draft pull request](https://github.com/PGBuildFarm/client-code/pull/38). You can also give it a review if you want :D
For testing the module, clone my fork and switch to the `abi-comp-check` branch. In build-farm.conf, put `modules => [qw(ABICompCheck)],` and test the module using the following command:
```bash
./run_build.pl --nosend --verbose --only-steps=make_testmodules
```

For the client, I initially thought the build farm server would also need to be set up locally for testing, but that wasn't the case. I'll cover the server setup in the next blog.

The client is implemented as a Perl module named `ABICompCheck.pm`(sorry for bad names I keep). 


### Things I had to figure out
**Disclaimer -** Although I have some programming experience, I'm not a Perl expert, and some parts of the build farm were written over 21 years ago (even before I was born lol), sooooo please correct me wherever you find something wrong in the code or implementation :)

#### 1. How to make a module for the build farm client.
Since I had to create a module, the first thing to figure out was how to make a module (obviously lol) that works with the PostgreSQL build farm. Thankfully, I found the documentation in the build farm docs, specifically '[Testing Additional Software](https://wiki.postgresql.org/wiki/PostgreSQL_Buildfarm_Howto#Testing_Additional_Software)'. It clearly specifies the presence of the Skeleton.pm file. This file is just a template for any module that already exists in the build farm.

The build farm client, as far as I can understand so far, runs modules procedurally in a series of hooks. The main script (i.e., `run_build.pl`) calls these hooks. Each module can define its own hooks, and the main script will call them in the order they are defined. This allows for a modular and extensible design. New functionality can be added by simply creating new modules and defining the appropriate hooks.

Currently I have the following three hooks for the `ABICompCheck.pm` module:
```perl
my $hooks = {
	'need-run' => \&need_run,
	'install' => \&install,
	'cleanup' => \&cleanup,
};
```
There's a `setup` subroutine that's meant for initializing the module with whatever configuration is required and registering the hooks for the module.

The `need_run` subroutine is responsible for determining whether the ABI compliance check needs to be run for that particular build.

The `install` subroutine contains the core logic for this module, which checks for any changes in ABI over the commits since last module run and generate the abidiff output if found any.

The `cleanup` subroutine is responsible for cleaning up the install, the PostgreSQL clone for this module, and also storing the git head hash for reference in the next run.

#### 2. How to use git for build farm
The build farm client has its own `SCM.pm` module, which is responsible for handling all git operations (and it also supports CVS in 2025!). After reading other existing PGBuild modules in the build farm, I learned that we can just use the `PGBuild::SCM` module. We create a new instance of it by passing our configuration like the repository, branch, etc., and then use its methods to perform git operations.

This might sound really simple, but the challenge I faced was that I needed to loop through commits for a particular branch in my module. However, the [`checkout`](https://github.com/PGBuildFarm/client-code/blob/b99f853c799ebbecc15962ad31d8a136cbe88a6b/PGBuild/SCM.pm#L1150) routine provided in the SCM module just copies the source code of the branch [without the .git folder](https://github.com/PGBuildFarm/client-code/blob/b99f853c799ebbecc15962ad31d8a136cbe88a6b/PGBuild/SCM.pm#L549). <br>This means I technically couldn't get the list of commits for that branch whose ABIs need to be compared. Additionally, the build farm client also maintains a local git mirror for its operations. So I decided to use the local git mirror, clone the branch from it using the `git clone` command. Then I similarly used other git commands like `git checkout` and `git rev-list` to perform further actions.

#### 3. Output Directory Structure
The `setup` subroutine creates all the required directories before starting the module execution. A `$binaries_rel_path` variable is defined to store the binary name after compilation, corresponding to its relative path in the `install` directory, as follows:
```perl
my $binaries_rel_path = $conf->{binaries_rel_path}
	  || {
		'postgres' => 'bin/postgres',
		'ecpg' => 'bin/ecpg',
		'libpq.so' => 'lib/libpq.so',
	  };
```
By default, the postgres and ecpg binaries, along with libpq.so, will be considered for ABI compliance check. The corresponding output directory structure of an initial run for two commits will look like:
```
./buildroot/abicheck.lucario/
├── diffs
│   └── postgres-9e345415bcd3c4358350b89edfd710469b8bfaf9-bf1119d74a79b68d9c9086e5d32d44fb294a1427.log
├── githead.log
├── logs
│   ├── 9e345415bcd3c4358350b89edfd710469b8bfaf9
│   │   ├── configure.log
│   │   ├── makeinstall.log
│   │   └── make.log
│   └── bf1119d74a79b68d9c9086e5d32d44fb294a1427
│       ├── configure.log
│       ├── makeinstall.log
│       └── make.log
└── xmls
    ├── 9e345415bcd3c4358350b89edfd710469b8bfaf9
    │   ├── ecpg.abi
    │   ├── libpq.so.abi
    │   └── postgres.abi
    └── bf1119d74a79b68d9c9086e5d32d44fb294a1427
        ├── ecpg.abi
        ├── libpq.so.abi
        └── postgres.abi
```
Here, the top-level folders obviously suggest what they're meant for. The sub-folders are named by commit hashes and contain the logs/XML files of the required binaries. The diffs folder is different as it contains only those logs where an ABI difference is actually detected by `abidiff`. The format is `<binary_name>-<commit_hash1>-<commit_hash2>.log`. The above file in the diffs folder is purely for representational purposes. Although I actually got an ABI change between some commits on the master branch, which is a good sign for the module :D

#### 4. Execution flow of the module
The main logic of the module is to compare the ABI of one previous commit to the next one.
- The `setup` subroutine initializes the module, creates the $abi_compare_root directory, and sets the $last_commit_hash variable from the file contents of the `githead.log` file.
- The `need_run` routine first clones from the local mirror in the `$abi_compare_root` directory.
- Checks out to `$pgbranch` - the branch currently being checked for ABI compliance
- Gets the `$head_commit_hash` for the current branch using `git rev-parse HEAD`
- Checks if `$last_commit_hash` is not an empty string
  - If yes, then compares the head commit hash with the last_commit_hash
	- If they are the same, then just returns as no ABI compliance check is needed.
	- If they are different, this means there are new commits since the last run of this module, so sets `$$run_needed=1`
  - If no - this means this is the first run of the module, so we need to run the ABI compliance check of the top two commits, sets `$$run_needed=1`
- Now, considering run_needed=1, the execution moves to the `install` subroutine.
- In the `install` subroutine, first changes directory to `$abi_compare_root/$clone_name` for performing operations on the source code
- If $last_commit_hash is empty
  - Gets the latest two commits using `git rev-list --max-count=2 HEAD` and sends the list of two commits to the `_process_commits_list` subroutine.
- Else
  - Gets a list of all commits after `$last_commit_hash` using `git rev-list --reverse $last_commit_hash...HEAD`. Sends the list to the `_process_commits_list` subroutine with the last_commit_hash as well. This is because this rev-list gives all commits after `$last_commit_hash` excluding the last_commit_hash itself.
- `_process_commits_list` subroutine loops through the list of commits and performs the following actions:
  - For each commit, it checks out the commit using `git checkout $commit_hash`
  - `_configure_make_and_build` the source code with `-g -Og` flags to get debug symbols required for libabigail tools.
  - `_generate_abidw_xml` using the `abidw` tool to generate the XML representation of compiled binaries.
  - `_compare_and_log_abi_diff` using the `abidiff` tool to compare the XML files of the current commit with the previous commit and log the ABI breakage if any.
- In the `cleanup` subroutine, it writes the current head commit hash to the `githead.log` file for the next run. It also cleans up the install directory and removes the cloned source code directory.

#### 5. Optimising _process_commits_list for a large list of commits - FastPath
The `_process_commits_list` subroutine is the most time-consuming part of the module, especially when there are a large number of commits to process.
The time that `_configure_make_and_build` takes is not under my control,<br>but what I can control is the looping over commits for the module. So, if the last commit hash is not empty, this means its XML files must have been generated in the previous run of the module and they must be stored in the `$abi_compare_root/xmls/$last_commit_hash` directory. Before moving to the next commit, I directly `_configure_make_and_build` the HEAD commit and get its XMLs as well. This way, I can compare the XMLs of the head commit with the last commit hash XMLs. If there's no difference, this means there must also be no ABI change in between. So I can skip all that processing, considering them to be good. This way, I can save a lot of time for the module execution.

## Why so late?
If you've noticed, my last blog was published on May 15, 2025, and it is now July 1, 2025.
I'm very new to blogging, and honestly, I procrastinated a little writing this one(although it was fun writing once I started), but I have been working on the project since then.
'No use crying over spilled milk' - I'll try to be more consistent from now on.

## Thanks for Reading!