# Git Notes

## Cross Platform Git Pitfalls

I recently encountered some problems using Dropbox to share a repository on both Mac and Windows (it's a little shady I know, but I found it practical in one particular workflow). I encountered some problems with both line ending characters (CRLF vs. LF) and file permissions (executable). These are my notes and solutions on the matter.

This note addresses two issues so far:

- Line endings
- File permissions

### Line Endings on Various Platforms and Git

Different OS uses different control characters to signal the ending of a line in text:

| OS            | Control Character Name | Control Characters | Hex   |
| ------------- | ---------------------- | ------------------ | ----- |
| Windows       | CRLF                   | \r\n               | 0D 0A |
| Linux / macOS | LF                     | \n                 | 0A    |

[Mind the End of Your Line](https://adaptivepatchwork.com/2012/03/01/mind-the-end-of-your-line/)  **MUST READ.** Historical walkthrough of how line endings are handling by Git. Details the old (global) vs. new (per repository) system of handling line endings.

Check your settings:

`Git config --list`

From SO[^autocrlf-values] (slightly edited for clarity)

```
╔═══════════════╦══════════════╦══════════════╦══════════════╗
║ core.autocrlf ║     false    ║     input    ║     true     ║
╠═══════════════╬══════════════╬══════════════╬══════════════╣
║   Git commit  ║ LF => LF     ║ LF => LF     ║ LF => CRLF   ║
║               ║ CRLF => CRLF ║ CRLF => LF   ║ CRLF => CRLF ║
╠═══════════════╬══════════════╬══════════════╬══════════════╣
║  Git checkout ║ LF => LF     ║ LF => LF     ║ LF => CRLF   ║
║  Git clone    ║ CRLF => CRLF ║ CRLF => CRLF ║ CRLF => CRLF ║
╚═══════════════╩══════════════╩══════════════╩══════════════╝
```

**false:**  Don't touch EOL.
**input:**  Only convert input to LF. No conversion to on checkout.
**true:**   Convert all commits to LF and checkout/clone to CRLF.

For more, see "Formatting and Whitespace" under [8.1 Customizing Git - Git Configuration](https://Git-scm.com/book/en/v2/Customizing-Git-Git-Configuration).

#### Quick Guide

Assuming:

1. `auto.crlf=true` which is the default on most Windows systems (in my experience), which means that all files in repository are LF
2. No `.Gitattributes` file.

##### Case 1: Repo NOT shared across platform in Dropbox

The defaults (assumptions above) should be fine. All platforms edit in their native EOL char, only LF is committed.

##### Case 2: Repo IS shared across platform in Dropbox

Enforce LF on both commit/checkout with:

1. Global command `Git config --global core.autocrlf input`
2. Local command `Git config --local core.autocrlf input`, which edits the .Git/config file.
3. Repository solution, add/edit `.Gitattributes` file with XX. Then clone repo again. For more see XX. One thing to note is that `* text=auto` seems unsafe because Git could accidentally format binary files, and if you do it manually, you need to add to `.Gitattributes` every time a new text file type is worked on.

Both 2 and 3 are good solutions.  **I chose solution 2**, because then my teammates are free to use the default CRLF editing locally if they want to, but solution 3 would probably be just as good, especially if everything in repo is already LF.

#### Detailed: What happens when sharing a repo in Dropbox across platforms?

Assuming:

1. `auto.crlf=true` which is the default on most Windows systems (in my experience), which means that all files in repo are LF
2. No `.Gitattributes` file.

##### Case 1: Repository is originally cloned/checked out on Windows (so all files are CRLF)

*All fine on windows, nothing will work on Linux/macOS.*

All is fine whether you commit or checkout. However on Linux/macOS, if this repository is opened, Git will want to convert all files to LF automatically, and will therefore show huge diffs for all files.

##### Case 2: Repository is originally cloned/checked out on Linux/macOS (so all files are LF)

*Commit is good on all platforms. Checkout on Windows goes badly.*

All is rosy as long as you're just committing on Windows. However if you checkout an old commit, all checked out files will be CRLF formatted. I would expect that you could either manually reformat the checked out CRLF files to LF or just discard changes, but none of it for me.

This one caused headache for me because I never checked out an earlier commit until far into the project, which suddenly gave me line ending issues.

##### Solutions to both cases

Enforce LF everywhere (see [Quick Guide](#case-2:-repo-is-shared-across-platform-in-dropbox))

#### `.Gitattributes` Settings

- `*.txt text` Set all files matching the filter `*.txt` to be text. This means that Git will run `CRLF` to `LF` replacement on these files every time they are written to the object database and the reverse replacement will be run when writing out to the working directory.
- `* test=auto`. When text is set to "auto", the path is marked for automatic end-of-line conversion. If Git decides that the content is text, its line endings are converted to LF on commit. When the file has been committed with CRLF, no conversion is done.[^Git-attributes]

The attributes allow a fine-grained control, how the line endings are converted. Here is an example that will make Git normalize .txt, .vcproj and .sh files, ensure that .vcproj files have CRLF and .sh files have LF in the working directory, and prevent .jpg files from being normalized regardless of their content[^attributes]

<!-- markdownlint-disable MD010 MD031 -->
```
*               text=auto
*.txt		text
*.vcproj	text eol=crlf
*.sh		text eol=lf
*.jpg		-text
```
<!-- markdownlint-enable MD010 MD031 -->

See also Git attributes [Effects](https://Git-scm.com/docs/Gitattributes#_effects). More on [core.eol](https://Git-scm.com/docs/Git-config#Git-config-coreeol) (which doesn't really need to be touched, see [Mind the End of Your Line](https://adaptivepatchwork.com/2012/03/01/mind-the-end-of-your-line/))

#### If LF and CRLF are already mixed

See GitHub Help article: [Dealing with line endings](https://help.Github.com/articles/dealing-with-line-endings/).

Optional additional reading - explains the same as [Mind the End of Your Line](https://adaptivepatchwork.com/2012/03/01/mind-the-end-of-your-line/) but differently.

| Article                                                      | tl;dr                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [What's the best CRLF (carriage return, line feed) handling strategy with Git?](https://stackoverflow.com/questions/170961/whats-the-best-crlf-carriage-return-line-feed-handling-strategy-with-Git) | Recommended read. One of the best SO questions on the issue. Good read for different opinions. |
| [You're just another carriage return line feed in the wall](https://www.hanselman.com/blog/YoureJustAnotherCarriageReturnLineFeedInTheWall.aspx) | People are still frustrated over this issue. "Why isn't this solved by sensible Git defaults??" |
| [WTF happened to my line endings?](https://gist.Github.com/shiftkey/b51f29301e52a3bc74d9) | Additional optional reading.                                 |
| [Force LF eol in Git repo and working copy](https://stackoverflow.com/questions/9976986/force-lf-eol-in-Git-repo-and-working-copy) | Additional optional reading.                                 |

[^autocrlf-values]: [Git on Windows: What do the crlf settings mean?](https://stackoverflow.com/questions/4181870/Git-on-windows-what-do-the-crlf-settings-mean)
[^attributes]: <https://Git-scm.com/docs/Gitattributes#Gitattributes-Settostringvaluelf>
[^Git-attributes]: <https://Git-scm.com/docs/Gitattributes#Gitattributes-Settostringvalueauto>

### File Permissions (executable) and Git

Sometimes Git will claim that files have changed when dealing with a repository cross platform, typically when starting it on Linux/macOS and copying it / using it on Windows. It can then manifest e.g. when checking out an older commit. Could look like this in Tower and VSCode:

![image-20180731083311323](assets/Git-file-permission-tower.png)

*How the problem looks in Tower.*

![Capture](assets/Git-file-permission-vscode.png)

*How the problem could look in VSCode. Note that no actual lines have changed, and the problem is not with line endings either.*

The problem and solution is described in medium article [How Git Treats Changes in File Permissions.](https://medium.com/@tahteche/how-Git-treats-changes-in-file-permissions-f71874ca239d), also Stack Overflow [How do I remove files saying "old mode 100755 new mode 100644" from unstaged changes in Git?](https://stackoverflow.com/questions/1257592/how-do-i-remove-files-saying-old-mode-100755-new-mode-100644-from-unstaged-cha).

In short: to fix, change Git config (use global option if desired):

`Git config --local core.filemode false`

As a user notes in a comment to top answer:

> This means that Git thinks that it can correctly set the executable bit on checked out files, but when it attempts to do so it doesn't work (or at least not in a way that it can read). When it then reads back the status of those files it looks like the executable bit has been deliberately unset. Setting core.filemode to false tells Git to ignore any executable bit changes on the filesystem so it won't view this as a change. If you do need to stage an executable bit change it does mean that you have to manually do `Git update-index --chmod=(+|-)x <path>` (- CB Baily)

PS: If the problem stems not from Unix vs. Windows disparity, you're probably better off [changing permissions](https://stackoverflow.com/a/44866012/2948823) with `chmod`.

## Tips

### gitignore: `**` matches any number of subfolders

The `**` wildcard matches any number of subdirectories. I.e. if you want to ignore all subfolders called `fig` within the `code` folder, add to gitignore: `code/**/fig`.

### Squash commits

The easy way to do it[^squash-commits]:

#### Write new commit message

```
git reset --soft HEAD~3
git commit
```

#### Squash all git commit messages

```
git reset --soft HEAD~3
git commit --edit -m"$(git log --format=%B --reverse HEAD..HEAD@{1})"
```

[^squash-commits]: https://stackoverflow.com/a/5201642/2948823
