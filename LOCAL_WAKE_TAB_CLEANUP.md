# Local Chrome wake-tab cleanup

This branch is based on AutoCLI `v0.3.8` and carries one local patch:

- wake Chrome with `http://localhost:<daemon-port>/ping#autocli-wake`;
- close only that exact sentinel tab after the extension WebSocket connects;
- preserve ordinary `about:blank`, `about:blank#blocked`, and user tabs.

Chrome rewrites externally opened `about:blank#autocli-wake` to
`about:blank#blocked`, so an HTTP URL served by the local daemon is used as the
marker instead.

## Test and build

Prerequisites: Rust stable, Node.js, and npm.

```sh
cargo test -p autocli-browser --lib

cd extension
npm ci
npx --yes vitest@5.0.5 run src/background.test.ts --reporter=verbose --no-file-parallelism
npm run build
cd ..

cargo build --release -p autocli
```

Known upstream `v0.3.8` baseline failures are outside this patch:

- `extension/src/cdp.test.ts`: `retries after cleanup when attach fails with a foreign extension error`;
- `autocli-pipeline`: `test_download_with_url_in_data`;
- extension typecheck: upstream does not declare Vitest types and is missing the
  `Window.__autocliSelectorActive` declaration.

## Install locally

Stop any running AutoCLI daemon before replacing the executable. Back up the
current files, then install the release binary and built extension script:

```sh
cp ~/.local/bin/autocli ~/.local/bin/autocli.backup-before-local-wake
cp ~/.local/share/autocli-extension/dist/background.js \
  ~/.local/share/autocli-extension/dist/background.js.backup-before-local-wake

cp target/release/autocli ~/.local/bin/autocli.new-local-wake
chmod 755 ~/.local/bin/autocli.new-local-wake
codesign --force --sign - ~/.local/bin/autocli.new-local-wake
mv ~/.local/bin/autocli.new-local-wake ~/.local/bin/autocli

cp extension/dist/background.js \
  ~/.local/share/autocli-extension/dist/background.js.new-local-wake
chmod 644 ~/.local/share/autocli-extension/dist/background.js.new-local-wake
mv ~/.local/share/autocli-extension/dist/background.js.new-local-wake \
  ~/.local/share/autocli-extension/dist/background.js
```

Open `chrome://extensions`, enable Developer mode, and click **Reload** on the
unpacked AutoCLI extension.

## Verify

With Chrome running:

```sh
autocli read https://example.com -f json
```

When the CLI prints `Waking up Chrome extension...`, the command should finish
without leaving any of these tabs behind:

- `http://localhost:19925/ping#autocli-wake`
- `about:blank`
- `about:blank#blocked`

## Roll back

Stop the AutoCLI daemon, restore both backups, and reload the unpacked Chrome
extension:

```sh
cp ~/.local/bin/autocli.backup-before-local-wake ~/.local/bin/autocli
chmod 755 ~/.local/bin/autocli
cp ~/.local/share/autocli-extension/dist/background.js.backup-before-local-wake \
  ~/.local/share/autocli-extension/dist/background.js
```

Do not delete the backups until the cold-start verification has passed.
