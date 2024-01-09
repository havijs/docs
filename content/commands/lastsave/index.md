---
acl_categories:
- '@admin'
- '@fast'
- '@dangerous'
arity: 1
command_flags:
- loading
- stale
- fast
complexity: O(1)
description: Returns the Unix timestamp of the last successful save to disk.
group: server
hidden: false
hints:
- nondeterministic_output
linkTitle: LASTSAVE
since: 1.0.0
summary: Returns the Unix timestamp of the last successful save to disk.
syntax_fmt: LASTSAVE
syntax_str: ''
title: LASTSAVE
---
Return the UNIX TIME of the last DB save executed with success.
A client may check if a [`BGSAVE`](/commands/bgsave) command succeeded reading the `LASTSAVE` value,
then issuing a [`BGSAVE`](/commands/bgsave) command and checking at regular intervals every N
seconds if `LASTSAVE` changed. Redis considers the database saved successfully at startup.