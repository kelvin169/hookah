# Hookah

[![Join the chat at https://gitter.im/hookah-server/Lobby](https://badges.gitter.im/hookah-server/Lobby.svg)](https://gitter.im/hookah-server/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![Go Report Card](https://goreportcard.com/badge/github.com/donatj/hookah)](https://goreportcard.com/report/github.com/donatj/hookah)
[![GoDoc](https://godoc.org/github.com/donatj/hookah?status.svg)](https://godoc.org/github.com/donatj/hookah)
[![Build Status](https://travis-ci.org/donatj/hookah.svg?branch=master)](https://travis-ci.org/donatj/hookah)

Hookah is a simple server for GitHub Webhooks that forwards the hooks message to any series of scripts, be they PHP, Ruby, Python or even straight up shell.

It simply passes the message on to the STDIN of any script.

## Installation

### From Source

Building v2 requires Go module support, available in Go 1.9.7+, 1.10.3+, 1.11+ or newer.

```bash
go get -u -v github.com/donatj/hookah/cmd/hookah
```

### From Binary

see: [Releases](https://github.com/donatj/hookah/releases).

## Basic Usage

When receiving a Webhook request from GitHub, Hookah checks `{server-root}/{vendor}/{repo}/{X-GitHub-Event}/*` for any ***executable*** scripts, and executes them sequentially passing the JSON payload to it's standard in.

This allows actual hook scripts to be written in any language you prefer.

For example, a script `server/donatj/hookah/push/log.rb` would be executed every time a "push" event Webhook was received from GitHub on the donatj/hookah repo.

## Example Hook Scripts

### bash + [jq](https://stedolan.github.io/jq/)

```bash
#!/bin/bash

set -e

json=`cat`
ref=$(<<< "$json" jq -r .ref)

echo "$ref"
if [ "$ref" == "refs/heads/master" ]
then
        echo "Ref was Master"
else
        echo "Ref was not Master"
fi

```

### PHP

```php
#!/usr/bin/php
<?php

$input = file_get_contents("php://stdin");
$data  = json_decode($input, true);

print_r($data);

```

### Note

Don't forget your scripts need to be executable. This means having the executable bit set ala `chmod +x <script filename>`, and having a [shebang](https://en.m.wikipedia.org/wiki/Shebang_(Unix)) pointing to your desired interpreter, i.e. `#!/bin/bash`

## Documentation

Standard input (stdin) contains the unparsed JSON body of the request.

### Execution

The server root layout looks like `{server-root}/{vendor}/{repo}/{X-GitHub-Event}/{script-name}`

Scripts are executed at each level, in order of least specific to most specific. At an individual level, the execution order is **file system specific** and *must not* be depended upon.

### Error Handling

Error handlers are scripts prefixed with `@@error.` and function similarly to standard scripts. Error handlers however are only triggered when the executiono of a normal script returns a **non-zero** exit code.

Error handlers like normal scripts trigger in order up from the root to the specificity level of the script.

### Example

Consider the following server file system.

```
├── @@error.rootlevel.sh
├── run-for-everything.sh
└── donatj
    ├── @@error.userlevel.sh
    ├── run-for-donatj-repos.sh
    └── hookah
        └── pull_request_review_comment
            ├── @@error.event-level.sh
            ├── likes-to-fail.sh
            └── handle-review.php
```

The execution order of a `pull_request_review_comment` event is as follows:

```
run-for-everything.sh
donatj/run-for-donatj-repos.sh
donatj/hookah/pull_request_review_comment/likes-to-fail.sh
donatj/hookah/pull_request_review_comment/handle-review.php
```

Now let's consider if `likes-to-fail.sh` lives up to it's namesake and returns a non-zero exit code. The execution order then becomes:

```
run-for-everything.sh
donatj/run-for-donatj-repos.sh
donatj/hookah/pull_request_review_comment/likes-to-fail.sh
@@error.rootlevel.sh
@@error.userlevel.sh
@@error.event-level.sh
donatj/hookah/pull_request_review_comment/handle-review.php
```

In contrast, imagining `donatj/run-for-donatj-repos.sh` returned a non-zero status, the execution would look as follows:

```
run-for-everything.sh
donatj/run-for-donatj-repos.sh
@@error.rootlevel.sh
@@error.userlevel.sh
donatj/hookah/pull_request_review_comment/likes-to-fail.sh
donatj/hookah/pull_request_review_comment/handle-review.php
```

### Environment Reference

#### All Executions

`GITHUB_EVENT` : The contents of the `X-Github-Event` header.

`GITHUB_DELIVERY` : The contents of the `X-GitHub-Delivery` header. A Unique ID for the Given Request

#### Error Handler Executions

`HOOKAH_EXEC_ERROR_FILE` : The path to the executable that failed to execute.

`HOOKAH_EXEC_ERROR` : The error message received while trying to execute the script.

`HOOKAH_EXEC_EXIT_STATUS` : The exit code of the script. This may **not** be defined in certain cases where execution failed entirely.
