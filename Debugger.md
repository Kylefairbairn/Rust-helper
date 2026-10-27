# Go Concurrency Debugging in VS Code

This guide sets up a Go project in Visual Studio Code and gives you a repeatable way to debug goroutines, data races, and hangs\.

## 1\. Install Go

Install Go using the [official installation guide](https://go.dev/doc/install)\. Then open a new VS Code terminal and check that Go is on your PATH:

```sh
go version
go env GOMOD
```

`go version` should print the installed version\. `go env GOMOD` prints the active module’s `go.mod` path &#40;or may report that no module is active yet&#41;\.

## 2\. Install VS Code Go tools

1. In Extensions, install **Go** by the Go Team\.
2. Open the Command Palette &#40;`Ctrl+Shift+P` on Windows/Linux or `Cmd+Shift+P` on macOS&#41;\.
3. Run **Go: Install/Update Tools**\.
4. Select at least:
   - `gopls` — Go editor features, navigation, and diagnostics\.
   - `dlv` — Delve, the Go debugger\.

The Go extension may offer to install or update Delve the first time you start a debug session\.

## 3\. Create or open a Go module

Open the project **folder** in VS Code\. In the integrated terminal, move to the project root\. If the project does not already have a `go.mod`, initialize one:

```sh
go mod init example.com/concurrency-demo
```

Use an appropriate module path if you plan to publish the repository\. Do not run `go mod init` again if the project already contains `go.mod`\.

Go’s standard library is available without installing separate modules\. When you add third\-party imports, Go resolves the dependencies\. To reconcile the module files with imports, run:

```sh
go mod tidy
```

This may create or update `go.sum`, which records dependency checksums\. Keep both `go.mod` and `go.sum` with the project\.

## 4\. Add a VS Code debug configuration

Create `.vscode/launch.json` in the project root:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Go package",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}"
    },
    {
      "name": "Debug Go tests",
      "type": "go",
      "request": "launch",
      "mode": "test",
      "program": "${workspaceFolder}"
    }
  ]
}
```

Set a breakpoint by clicking in the gutter next to a line number\. Open **Run and Debug**, select a configuration, and press **F5**\. Use **F10** to step over a line, **F11** to step into a function, and **Shift\+F11** to step out\.

For goroutines, inspect the **Call Stack** section in the Run and Debug view\. Select a goroutine to see its stack and variables\. Resuming or pausing in the debugger affects the running program and can change the timing of a concurrency bug\.

## 5\. Run Go’s race detector

Run the code with race detection enabled:

```sh
go run -race .
```

For packages with tests:

```sh
go test -race ./...
```

If the race detector prints `WARNING: DATA RACE`, it found concurrent accesses to shared memory where at least one access is a write and the accesses are not properly synchronized\. Read both reported stack traces to find the conflicting accesses\. The detector only finds races on code paths that actually ran\.

## 6\. Tell a race from a hang

- **Race:** values are occasionally wrong, results vary between runs, or `-race` reports conflicting reads and writes\.
- **Deadlock / blocked goroutines:** the program stops making progress\. Pause it in the debugger and inspect the goroutine stacks for waits on channels, mutexes, or a `WaitGroup`\.
- **Goroutine leak:** the program keeps running and the number of goroutines grows because some work is never stopped or drained\.

Common synchronization tools:

- `sync.Mutex` protects shared mutable data\.
- Channels pass values or signal work between goroutines\.
- `sync.WaitGroup` waits for a group of goroutines to finish\. Call `Add` before starting the goroutine and `Done` when it exits &#40;often with `defer`&#41;\.
- `sync/atomic` supports specific atomic operations; it does not make arbitrary shared data safe\.

A mutex or channel should protect the data or operation that actually needs coordination\. Adding a lock without identifying the shared state can hide symptoms without fixing the design\.

## 7\. Useful commands

Run these from the module root:

```sh
go run .
go test ./...
go test -race ./...
go vet ./...
go mod tidy
```

- `go run .` builds and runs the current package\.
- `go test ./...` runs tests in this module and its subdirectories\.
- `go test -race ./...` runs those tests with race detection\.
- `go vet ./...` reports suspicious code patterns\.
- `go mod tidy` updates module dependencies based on imports\.

## If debugging still does not start

1. Confirm `go version` works in the VS Code integrated terminal\.
2. Confirm VS Code has the project folder open and that `go.mod` is at its root\.
3. Run **Go: Locate Configured Go Tools** from the Command Palette and confirm the extension can find `dlv`\.
4. Run **Go: Install/Update Tools** and update `gopls` and `dlv`\.
5. Check the **Output** panel’s **Go** channel for extension or debugger errors\.

To diagnose a specific concurrency issue, capture the relevant goroutine code, the exact command you ran, and the complete race\-detector output or the point where the program hangs\.
