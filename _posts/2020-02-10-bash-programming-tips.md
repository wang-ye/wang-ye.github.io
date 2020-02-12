A couple of tips for bash programming debugging. This is a WIP.

First, fail fast.

```bash
#!/bin/bash
set -o nounset
set -o errexit
```

For debugging, use

```bash
set -o verbose
set -o xtrace
```

bash -n to check syntax, and bash -x to run every statement in the scripts.