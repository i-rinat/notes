---
layout: post
title:  "mysql-test-run and rr"
date:   2021-11-26 21:02:03 +0300
tags:   mysql rr debug
---

[Mysql-test-run](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_TEST_RUN_PL.html)
(aka MTR or `mysql-test-run.pl`) supports various debuggers on
various platforms, but it doesn't support `rr` at the moment. The easiest
solution is to hijack code path for `ddd` support:

```diff
--- a/mysql-test/mysql-test-run.pl
+++ b/mysql-test/mysql-test-run.pl
@@ -7393,6 +7393,11 @@ sub ddd_arguments {
   my $type  = shift;
   my $input = shift;
 
+  unshift @$$args, $$exe;
+  unshift @$$args, "record";
+  $$exe = "/usr/bin/rr";
+  return;
+
   my $gdb_init_file = "$opt_vardir/tmp/gdbinit.$type";
 
   # Remove the old gdbinit file
```

Then, append `--ddd` to MTR's command line.

To run `rr` in a Docker container, additional permissions are needed:
`--cap-add=SYS_PTRACE --security-opt seccomp=unconfined`
([link](https://github.com/rr-debugger/rr/wiki/Docker)).
