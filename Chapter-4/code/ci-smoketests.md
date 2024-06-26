# Code for smoketests during CI

```yaml
name: Run a smoke test (CI)

# The smoke test runs on *any* push to the repo and can also be triggered manually.
# It does a simple build and test using Go toolchain directly in the GitHub runner.
on:
  push:
  workflow_dispatch:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          persist-credentials: false

      - name: Install Go
        uses: actions/setup-go@v3
        with:
          go-version-file: go.mod

      - name: Fetch required Go modules
        run:  go mod download

      - name: Build
        run:  go build -v ./...

      - name: Test
        run:  go test -v ./...
```
