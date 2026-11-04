---
type: llm
weight: 1
---

PASS requires all of the following:

1. The key check runs a concrete framedash subcommand that needs no project id: framedash auth or framedash projects list. A project-scoped command such as framedash status or framedash dashboard counts only if the response also gives it a project id, because it exits before the request otherwise.
2. The response recommends a safe key path: --api-key-file pointing at a key file, --api-key-file - reading the key from stdin, or FRAMEDASH_API_KEY supplied by the CI runner or a secret store rather than typed at the shell.
3. The dashboard numbers come from framedash dashboard.
4. That dashboard call itself carries the project id -- --project-id <uuid> or FRAMEDASH_PROJECT_ID -- because framedash dashboard exits nonzero without one. Naming framedash projects list or framedash auth as the way to discover the id strengthens a PASS but does not replace passing it to the call.
5. The response accounts for JSON output: --format json, FRAMEDASH_FORMAT=json, or stating that json is the CLI default output format.

FAIL if any of the five is missing, if the response puts the raw key on the command line as --api-key with no warning that it lands in shell history and the process list, if its safe path is a literal FRAMEDASH_API_KEY assignment typed at the shell (which lands in history the same way), if it invents a command that is not a Framedash CLI command, or if it never names the framedash binary.
