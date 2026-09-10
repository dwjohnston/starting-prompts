````markdown
## Tests that create external resources

**Assume that the test runner may be terminated at any time.**

For example, a user may press `Ctrl+C`, sending `SIGINT` to the test process. The process may also receive `SIGTERM`, or it may be terminated in a way that does not allow cleanup code to run.

External resources include, but are not limited to:

- Child processes
- Docker containers
- Temporary services
- Listening ports
- Temporary files and directories
- Other resources with an externally visible lifecycle

**Never rely solely on `after`, `afterEach`, or `afterAll` hooks to clean up external resources.**

### Check required resources before starting

Before creating any external resources, use a `beforeAll` hook to verify that resources required by the test suite are available.

For example, if tests require specific ports, verify that **all required ports are free before starting any processes or containers**.

If a required resource is already in use, immediately fail the tests with a helpful, actionable error.

**Do not attempt to kill or remove an existing resource**, since it may belong to the user or another application.

For example:

```typescript
import { beforeAll } from "bun:test";

beforeAll(async () => {
  const ports = [3000, 3001];

  for (const port of ports) {
    const available = await isPortAvailable(port);

    if (!available) {
      throw new Error(
        `Cannot run tests: port ${port} is already in use. ` +
        `A previous test run or another application may still be running. ` +
        `Stop the process using port ${port} and run the tests again.`,
      );
    }
  }
});
```

Apply the same principle to other exclusive resources where practical, such as Docker container names, networks, or volumes.

### Use a shared cleanup mechanism

Create a single cleanup mechanism that knows about **all external resources created by the test suite**.

For child processes, use an `AbortController`:

```typescript
const shutdown = new AbortController();

function spawn(cmd: string[]) {
  return Bun.spawn({
    cmd,
    signal: shutdown.signal,
  });
}
```

Docker containers and other resources must also be tracked and removed by the cleanup mechanism.

**Register every external resource for cleanup as soon as it is created.**

Where possible, creation helpers should register resources before returning them to the caller. This reduces the possibility of interruption occurring between creation and registration.

### Clean up on normal completion and termination

Run cleanup both during normal test completion and when the test runner receives termination signals.

```typescript
afterAll(cleanup);

process.on("SIGINT", () => {
  cleanup();
  process.exit(130);
});

process.on("SIGTERM", () => {
  cleanup();
  process.exit(143);
});
```

Cleanup should terminate child processes and remove Docker containers and other external resources created by the current test run.

Cleanup on termination is **best-effort**. Some forms of termination, such as `SIGKILL`, cannot be intercepted.

Therefore, the test suite must also be safe to run after a previous test run was forcibly terminated.

### Make resources identifiable

Where possible, give external resources a unique test-run identifier and/or metadata identifying them as belonging to the test suite.

For Docker resources, prefer unique names and labels rather than generic names.

For example:

```text
my-test-<run-id>-postgres
```

Unique identifiers make leftover resources easier to diagnose and prevent tests from accidentally interacting with resources belonging to another test run or the user.

**Never blindly kill or remove an existing resource merely because it has the expected name or uses an expected port.**
````
