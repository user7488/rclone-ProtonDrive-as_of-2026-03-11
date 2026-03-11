# How to Compile rclone with the ProtonDrive Upload Fix

## Prerequisites

- Go 1.25+ (`go version`)
- All three repos cloned side-by-side under the same parent directory:
  ```
  /$HOME/
  ├── go-proton-api/          # github.com/rclone/go-proton-api
  ├── Proton-API-Bridge/      # github.com/rclone/Proton-API-Bridge
  └── rclone/                 # github.com/rclone/rclone
  ```

## 1. Checkout the fix branches

```bash
cd go-proton-api
git checkout fix/upload-verification-token

cd ../Proton-API-Bridge
git checkout fix/upload-verification-token

cd ../rclone
git checkout fix/protondrive-upload-422
```

## 2. Set up `replace` directives in `go.mod`

The library dependencies form a chain: rclone → Proton-API-Bridge → go-proton-api.
Each downstream repo needs a `replace` directive to use the local copy.

### Proton-API-Bridge `go.mod`

Add at the end (use your actual absolute path):

```bash
cd ../Proton-API-Bridge
cat >> go.mod << 'EOF'

replace github.com/rclone/go-proton-api => ../go-proton-api
EOF
```

### rclone `go.mod`

Add at the end (use your actual absolute paths):

```bash
cd ../rclone
cat >> go.mod << 'EOF'

replace github.com/rclone/go-proton-api => ../go-proton-api
replace github.com/rclone/Proton-API-Bridge => ../Proton-API-Bridge
EOF
```

> **Note:** If rclone's `go.mod` already has `replace` directives for these
> modules (e.g. pointing to `-local` dirs), edit them to point to the correct
> paths instead of appending.

## 3. Build

```bash
cd rclone
go build -o rclone .
```

The binary is at `./rclone`. Verify:

```bash
./rclone version
```

## 4. Test the fix

Quick upload test:

```bash
./rclone copy somefile.txt TestProtonDrive:test-folder/
```

Run the integration test suite:

```bash
go test -v -run "TestIntegration/FsMkdir/FsPutFiles" -count=1 -timeout 180s ./backend/protondrive/
```

## For Publishing (after PR merge)

Once the go-proton-api and Proton-API-Bridge changes are merged and tagged upstream:

1. Remove the `replace` directives from both `go.mod` files
2. Update the module versions:
   ```bash
   cd Proton-API-Bridge
   go get github.com/rclone/go-proton-api@<new-tag>
   go mod tidy

   cd ../rclone
   go get github.com/rclone/go-proton-api@<new-tag>
   go get github.com/rclone/Proton-API-Bridge@<new-tag>
   go mod tidy
   ```
3. Commit the updated `go.mod` / `go.sum` files
