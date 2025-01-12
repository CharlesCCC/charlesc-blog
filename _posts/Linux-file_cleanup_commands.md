---
title: "How to clean up files in Linux"
date: "2025-01-12"
author:
  name: Charles
  picture: "/assets/blog/authors/cc.png"
categories: 
  - "instruction"
coverImage: "/assets/blog/hello-world/cover.jpg"
ogImage:
  url: "/assets/blog/preview/cover.jpg"
---

# Linux File Cleanup Commands

## Remove Files Older Than 24 Hours

Here are various commands to remove files older than 24 hours:

1. Using `find` command with `-mtime`:
```bash
find /path/to/directory -mtime +1 -type f -delete
```

2. Using `find` with `-exec rm`:

```bash
find /path/to/directory -mtime +1 -type f -exec rm {} \;
```

3. Using `find` with `-ctime` (based on change time):
```bash
find /path/to/directory -ctime +1 -type f -delete
```

4. Using `-mmin` for more precise control (1440 minutes = 24 hours):
```bash
find /path/to/directory -mmin +1440 -type f -delete
```

### Explanation of parameters:
- `/path/to/directory`: Replace with your target directory path
- `-mtime +1`: Files modified more than 1 day ago
- `-ctime +1`: Files changed more than 1 day ago
- `-mmin +1440`: Files modified more than 1440 minutes ago
- `-type f`: Only match files (not directories)
- `-delete`: Delete the matched files

### Safety tip: 
Before deleting, you can remove the `-delete` option to see which files would be deleted:
```bash
find /path/to/directory -mtime +1 -type f
```

## Count Total Files

Various methods to count files from a `find` command:

1. Using `wc -l`:
```bash
find /path/to/directory -mtime +1 -type f | wc -l
```

2. Using `find` with `-printf` and `wc`:
```bash
find /path/to/directory -mtime +1 -type f -printf '.' | wc -c
```

3. Using a counter variable in combination with `find`:
```bash
count=$(find /path/to/directory -mtime +1 -type f | wc -l)
echo "Total files: $count"
```

4. Using `-exec` with a counter:
```bash
find /path/to/directory -mtime +1 -type f -exec echo \; | wc -l
```

### Note:
The most common and simplest approach is using `wc -l`. Here:
- `wc -l` counts the number of lines in the output
- Each line represents one file found by the `find` command