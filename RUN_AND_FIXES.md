# Run & Fixes - Blockchain Game

This document records the Podman + Playwright testing session, bugs found, fixes applied, and how to run the project now.

## 1. How the project was tested

- Built and ran the whole stack in Podman via `container/Containerfile` + `container/start.sh` (see `podman build` / `podman run --network host` below).
- Verified services from host: `curl http://localhost:3000` (client), `curl http://localhost:8545` (Hardhat `eth_chainId` → `0x7a69` = 31337), `curl http://localhost:9208/signer` → trusted `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`.
- Drove the browser with Playwright (`chrome-for-testing`) at `http://localhost:3000`:
  - Checked `[Scene] StartScene created` / `ready` (`client/src/scenes/startScene.ts:91`, `145`), `Phaser v3.55.2`.
  - Clicked the `CONNECT WALLET` canvas button via `page.mouse` + checked console logs (`[Button] Connect button clicked` etc. per `WALLET_TEST_CHECKLIST.md`).
  - Followed the wallet → `login` → `authenticate()` → `CONNECT_SCENE` → `geckos` flow and inspected console + `Fetch`/`WebSocket` errors.

## 2. Bugs found

### a) `CONNECT WALLET` not clickable - Phaser / phaser3-nineslice drift
- `client/package.json:29` allowed `^3.55.2`, lockfile resolved `3.90.0`. `phaser3-nineslice@0.5.0` (`client/node_modules/phaser3-nineslice/dist/nineslice.js:1`) subclasses `Phaser.GameObjects.RenderTexture` which was rewritten in 3.60+ (`client/node_modules/phaser/src/gameobjects/rendertexture/RenderTexture.js:48` now an `Image` shim over `DynamicTexture`).
- Result on `3.90`: `this.width === "btngrey"`, `displayWidth === null`, `input.hitArea === undefined` (`client/src/scenes/startScene.ts:104` `add.nineslice(...).setInteractive()` produced no hitArea) → `hitTest` always missed. Console showed `Texture "__MISSING" has no frame "__MISSING" x18` and canvas button rendered but never received `pointerup`.

### b) Canvas `y = 931` offset - loading overlay
- `client/index.html:16` `#loading { height:100vh }` was in-flow. `new Phaser.Game()` in `client/src/main.ts:10` is constructed before `StartScene.create:94` hides `#loading`. At construction `ScaleManager.getParentBounds()` measured body with `#loading` (931px) + canvas (931px) stacked → `scale.canvasBounds.y = 931` → Playwright `page.mouse` at `(456,465)` mapped to `pointer.y = -466` → `worldY 0` → missed even after (a) was fixed. Seen via `window.__GAME__.scale.canvasBounds` and `g.input.mousePointer`.

### c) `autoCenter` with `RESIZE`
- `client/src/main.ts:14` `autoCenter: Phaser.Scale.CENTER_BOTH` is redundant with `mode: RESIZE` (canvas already `= parentSize`) and contributed to stale bounds handling. Should be `NO_CENTER` for `RESIZE`.

### d) Geckos URL missing scheme
- `client/src/scenes/connectScene.ts:25` `host = "localhost"` → `geckos({url:"localhost", port:9208})` → `fetch("localhost:9208/.wrtc/v2/connections")` → `Fetch API cannot load localhost:9208/... URL scheme "localhost" is not supported` (`@geckos_io_client.js:358`, `connectScene.ts:36` `Failed to fetch`). `client/src/scenes/startScene.ts:215` already uses `http://localhost:9208`.

### e) `server/server.js:120` silent start
- Original logged via `FileEditor` watcher, no `listening on http://localhost:9208` line, harder to health-check in Podman.

### f) Docker build cache clobber
- No `.dockerignore` at context root (only `container/.dockerignore:1`). `COPY client ./client` (`container/Containerfile:15`) copied host `client/node_modules` (still `3.90`) over the image's fresh `npm ci` `3.55.2`, re-introducing (a) in the built image. Verified `podman run --rm ... node -e "require('/app/client/node_modules/phaser/package.json').version"` showed `3.90` despite `package.json`/`package-lock.json` being `3.55.2`.

## 3. Changes made

| File | Line | Change |
|------|------|--------|
| `client/package.json:29` | `phaser` | `^3.55.2` → `3.55.2` (pin) |
| `client/package-lock.json:1` | `phaser` | regenerated (`npm install`) → `3.55.2` (`packages["node_modules/phaser"].version`) |
| `client/src/main.ts:14` | `scale.autoCenter` | `CENTER_BOTH` → `NO_CENTER` |
| `client/index.html:16` | `#loading` | `position:absolute; top:0; left:0; width:100%; height:100%; background:#171717; z-index:10` instead of in-flow `height:100vh` |
| `client/src/scenes/connectScene.ts:25` | `host` | `... ? VITE_HOST : "localhost"` → `... ? VITE_HOST.replace(/:\d+\/?$/, "") : "http://localhost"` (adds `http://`, strips port if env already contains it) |
| `server/server.js:9` | imports | removed `spawn` + `FileEditor`, added `console.log('listening on http://localhost:9208')` on `server.listen` |
| `.dockerignore` (new at repo root) | - | copied from `container/.dockerignore` (`**/node_modules` etc.) so `podman build` from `.` context doesn't copy host `node_modules` |

Also ran `cd client && npm install` on host to sync `client/node_modules` to `3.55.2`.

## 4. How to run now

### Podman (recommended - single command, as tested)

```bash
# from repo root
podman build -f container/Containerfile -t localhost/blockchain-game:test .
podman run -d --name bgame-test --network host -e PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 localhost/blockchain-game:test
# wait ~20s until Vite ready
curl -s http://localhost:3000/ | grep -q "Blockchain Game" && echo "client up"
curl -s http://localhost:9208/signer  # → 0xf39Fd6e...
curl -s -X POST -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' http://localhost:8545
podman logs --tail 20 bgame-test
# stop
podman stop bgame-test && podman rm bgame-test
```

`container/start.sh:1` does: flatten `ClaimVerifier`/`ClaimManagerERC721` → `src/flat`, `hardhat node --hostname 0.0.0.0`, wait RPC, `hardhat compile + deploy --network localhost`, `node server.js` (`:9208`), `vite --host 0.0.0.0 --port 3000`.

Image exposes `3000 8545 9208` but container is run with `--network host` so they are `localhost:3000`, `localhost:8545`, `localhost:9208` on the host (needed for MetaMask + geckos WebRTC).

### Manual (4 terminals, no Podman)

```bash
git clone ... && cd blockchain-game
npm install; cd server && npm install && cd ..; cd client && npm install && cd ..
# T1
npm run node      # contracts/hardhat node
# T2
npm run deploy    # hardhat deploy --network localhost
# T3
npm run server    # server/nodemon server.js → http://localhost:9208
# T4
npm run client    # client/vite → http://localhost:3000
```

### MetaMask for login (fixes `commons/auth.mjs:5` chainId 31337)

- Add network: `Hardhat Local` → RPC `http://localhost:8545`, Chain ID `31337` (`0x7a69`), Symbol `ETH`.
- If you saw `Provided chainId "31337" must match the active chainId "1"` (`client/src/scenes/startScene.ts:243`), switch to that network and refresh.
- Quick console alternative:
  ```js
  await window.ethereum.request({method:'wallet_switchEthereumChain', params:[{chainId:'0x7a69'}]}).catch(()=>window.ethereum.request({method:'wallet_addEthereumChain', params:[{chainId:'0x7a69', chainName:'Hardhat Local', rpcUrls:['http://localhost:8545'], nativeCurrency:{name:'ETH',symbol:'ETH',decimals:18}}]}))
  ```
- At `http://localhost:3000`: `CONNECT WALLET` → pick **MetaMask** (`startScene.ts:14` `MetaMask available: true`) → Approve → button becomes `login` → Sign the `Challenge` typed data (`_signTypedData` `domain:{name:"blockchain game",version:"1",chainId:31337}`) → `Authentication successful, switching to CONNECT_SCENE` (`startScene.ts:240`) → `logging in to server...` (`connectScene.ts:23`) → `geckos` connects to `http://localhost:9208` (`connectScene.ts:28`) → dungeon. Collect the coin to test `Trustus` claim (`ClaimVerifier`/`ClaimManagerERC721`).

## 5. Verification after fixes (Podman + Playwright)

- `Phaser v3.55.2` (was `3.90.0`), no `__MISSING` texture warnings, only `willReadFrequently` + harmless `Expected less than 7 arguments for NineSlice creation, received 9.` (`phaser3-nineslice` 9-arg call).
- `page.mouse` at canvas center now logs `[Button] Connect button clicked` → `[Wallet] Starting wallet connection...` → `MetaMask available: false` (headless) → `Web3Modal` → `POST http://localhost:9208/challenge` → `0x...` challenge, signer `0x5122...` etc. After host `http://` fix, `geckos` fetch uses `http://localhost:9208/.wrtc/v2/connections` (was `localhost:9208/...`).
- In headed browser with MetaMask on `31337`, full flow is `Signer obtained` → `Got challenge` → `Generated typed data` → `Signature obtained` → `CONNECT_SCENE`.

## 6. Rebuilding

Any change to `client/*`, `server/*`, `contracts/*`, `commons/*` needs:

```bash
podman build -f container/Containerfile -t localhost/blockchain-game:test .
```

Root `.dockerignore` ensures host `node_modules` isn't copied and stale `3.90` doesn't return.
