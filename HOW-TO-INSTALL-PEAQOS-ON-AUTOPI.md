# How to install peaqOS on AutoPi

Official install: [docs.peaq.xyz/peaqos/install](https://docs.peaq.xyz/peaqos/install). That path is `pip install peaq-os-cli` on 64-bit with Python 3.10+. This file is the workaround for the stock AutoPi.

Stock AutoPi is Raspbian 10, 32-bit armv7, about 1 GB RAM, Python 3.7. peaqOS wants Python 3.10+. `pip install peaq-os-cli` will not work on that image.

What we ended up with: peaq-os-cli 0.0.7 and peaq-os-sdk 0.5.0, running in a venv. AutoPi’s own Python stays 3.7.

If your board is 64-bit and already has Python 3.10+, ignore this file and just do:

```bash
python3 -m venv .peaq-os
source .peaq-os/bin/activate
pip install "peaq-os-cli==0.0.7"
peaqos --version
```

The rest is for the 32-bit AutoPi.

Connect to the device first — see AutoPi’s own docs: [How to SSH to Your Device](https://docs.autopi.io/developer_guides/how-to-ssh-to-your-device/). You need internet on the unit and a bit of free disk.

The commands below use `/home/pi/peaqos` as an example install folder. Any writable path is fine; just keep it consistent. `peaqos` never lands on PATH — you call the binary in the venv.

We used [uv](https://docs.astral.sh/uv/) to get Python 3.11 without touching `/usr/bin/python3`. It’s just a tool to install Python and packages. Not part of peaqOS.

## Python 3.11 in a venv

```bash
mkdir -p /home/pi/peaqos/tools
cd /home/pi/peaqos
curl -LsSf https://astral.sh/uv/install.sh | env UV_INSTALL_DIR="/home/pi/peaqos/tools" UV_NO_MODIFY_PATH=1 sh
export PATH="/home/pi/peaqos/tools:$PATH"

uv python install 3.11
uv venv /home/pi/peaqos/.peaq-os --python 3.11
```

`uv python install` also drops files under `~/.local/share/uv`. That’s normal.

## peaqOS packages

Don’t let pip pull all dependencies. `google-re2` has no armv7 wheel and it won’t compile.

```bash
export PATH="/home/pi/peaqos/tools:$PATH"

uv pip install --python /home/pi/peaqos/.peaq-os/bin/python \
  --no-deps "peaq-os-cli==0.0.7" "peaq-os-sdk==0.5.0"
```

cryptography 50.x tries to build from source and dies on Buster. Use 44.0.1 (there’s an armv7 wheel):

```bash
WHL=/home/pi/peaqos/tools/cryptography-44.0.1-cp39-abi3-manylinux_2_28_armv7l.whl

curl -L --fail -o "$WHL" \
  "https://files.pythonhosted.org/packages/e6/50/bf8d090911347f9b75adc20f6f6569ed6ca9b9bff552e6e390f53c2a1233/cryptography-44.0.1-cp39-abi3-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl"

uv pip install --python /home/pi/peaqos/.peaq-os/bin/python "$WHL"
```

Then the rest. Skip Pillow — the image doesn’t have jpeg headers.

```bash
uv pip install --python /home/pi/peaqos/.peaq-os/bin/python \
  "click>=8.1" "python-dotenv>=1.0" \
  "eth-account>=0.10,<1.0" "PyYAML>=6.0" \
  "web3>=6.0" requests "posthog>=3.0" \
  "pycryptodome>=3.20.0" "PyNaCl>=1.5.0"
```

## Two small shims

The SDK imports `re2` (google-re2). Stdlib `re` is enough for whoami / activate / hash / sign.

Pillow is stubbed so the CLI can import `PIL`.

```bash
SITE=$(/home/pi/peaqos/.peaq-os/bin/python -c 'import site; print(site.getsitepackages()[0])')

cat > "$SITE/re2.py" <<'PY'
import re

error = re.error


class Options(object):
    def __init__(self):
        self.log_errors = False


def compile(pattern, options=None):
    return re.compile(pattern)


def search(pattern, string):
    return re.search(pattern, string)


def fullmatch(pattern, string):
    return re.fullmatch(pattern, string)
PY

mkdir -p "$SITE/PIL"
printf '%s\n' 'class Image(object):' '    pass' > "$SITE/PIL/Image.py"
printf '%s\n' '# stub — no jpeg headers on AutoPi' > "$SITE/PIL/__init__.py"
```

## .env (Agung testnet)

CLI walks up from site-packages looking for `.env`, so put it at `/home/pi/peaqos/.env`.

MetaMask keys need the `0x` prefix (0x + 64 hex).

```bash
cat > /home/pi/peaqos/.env <<'ENV'
PEAQOS_PRIVATE_KEY=0xYOUR_64_HEX_CHARS_HERE
PEAQOS_RPC_URL=https://peaq-agung.api.onfinality.io/public
PEAQOS_MCR_API_URL=https://mcr-20.peaq.xyz
PEAQOS_NETWORK=testnet
PEAQOS_ORCHESTRATION_URL=https://orchestration.peaq.xyz
PEAQOS_ORCH_API_KEY=

IDENTITY_REGISTRY_ADDRESS=0x9E9463a65c7B74623b3b6Cdc39F71be7274e5971
IDENTITY_STAKING_ADDRESS=0x55f336714aDb0749DbFE33b057a1702405564E3d
EVENT_REGISTRY_ADDRESS=0x2DAD8905380993940e340C5cE6d313d5c2780040
MACHINE_NFT_ADDRESS=0xB41C2A4f1c19b6B06beaAce0F5CD8439e77C4b1c
DID_REGISTRY_ADDRESS=0x0000000000000000000000000000000000000800
BATCH_PRECOMPILE_ADDRESS=0x0000000000000000000000000000000000000805
ENV
chmod 600 /home/pi/peaqos/.env
```

For peaq mainnet, use this instead:

```bash
cat > /home/pi/peaqos/.env <<'ENV'
PEAQOS_PRIVATE_KEY=0xYOUR_64_HEX_CHARS_HERE
PEAQOS_RPC_URL=https://quicknode1.peaq.xyz
PEAQOS_MCR_API_URL=https://mcr-20.peaq.xyz
PEAQOS_NETWORK=mainnet
PEAQOS_ORCHESTRATION_URL=https://orchestration.peaq.xyz
PEAQOS_ORCH_API_KEY=

IDENTITY_REGISTRY_ADDRESS=0xb53Af985765031936311273599389b5B68aC9956
IDENTITY_STAKING_ADDRESS=0x11c05A650704136786253e8685f56879A202b1C7
EVENT_REGISTRY_ADDRESS=0x43c6AF2E14dc1327dc3cc6c7117D1CD72fffEcbA
MACHINE_NFT_ADDRESS=0x2943F80e9DdB11B9Dd275499C661Df78F5F691F9
DID_REGISTRY_ADDRESS=0x0000000000000000000000000000000000000800
BATCH_PRECOMPILE_ADDRESS=0x0000000000000000000000000000000000000805
ENV
chmod 600 /home/pi/peaqos/.env
```

Fallback mainnet RPCs: `https://quicknode2.peaq.xyz`, `https://peaq.api.onfinality.io/public`. On mainnet, the bond and gas are paid in real PEAQ.

## Check

```bash
cd /home/pi/peaqos
.peaq-os/bin/peaqos --version
# peaq-os-cli 0.0.7 (peaq_os_sdk 0.5.0)

.peaq-os/bin/python --version
# Python 3.11.x

.peaq-os/bin/peaqos whoami
```

whoami hits the Agung RPC. Give it a minute.

## Activate

Needs ~1.2 AGNG (1 AGNG bond + gas).

```bash
cd /home/pi/peaqos
.peaq-os/bin/peaqos activate --skip-funding
```

The ID is tied to the wallet, not the hardware. New key = new ID and another bond.

The machine address and the activation txs show up on [agung-testnet.subscan.io](https://agung-testnet.subscan.io). Search the address, or open them directly:

```
https://agung-testnet.subscan.io/account/<machine-address>
https://agung-testnet.subscan.io/tx/<tx-hash>
https://agung-testnet.subscan.io/evm_nft/<IDENTITY_REGISTRY_ADDRESS>/<machine-id>
```

On peaq mainnet, the same pages are on [peaq.subscan.io](https://peaq.subscan.io):

```
https://peaq.subscan.io/account/<machine-address>
https://peaq.subscan.io/tx/<tx-hash>
https://peaq.subscan.io/evm_nft/<IDENTITY_REGISTRY_ADDRESS>/<machine-id>
```

`IDENTITY_REGISTRY_ADDRESS` is the one in `.env`.

Our AutoPi (`autopi-8768c64b7660b2d974feaeb1cd4b5bf2`) is machine **#347** on Agung, for reference:

- Wallet: [0x1DD3…7F90](https://agung-testnet.subscan.io/account/0x1DD3e3b78C0fFb25E0Fd1e17751a4C2061b07F90) · [peaqscan](https://testnet.peaqscan.xyz/address/0x1DD3e3b78C0fFb25E0Fd1e17751a4C2061b07F90)
- Identity NFT #347: [IdentityRegistry token 347](https://agung-testnet.subscan.io/evm_nft/0x9e9463a65c7b74623b3b6cdc39f71be7274e5971/347)
- Machine NFT token 41: [MachineNFT token 41](https://agung-testnet.subscan.io/evm_nft/0xb41c2a4f1c19b6b06beaace0f5cd8439e77c4b1c/41)
- Activate txs: [mint](https://agung-testnet.subscan.io/tx/0xa51bad9896976bea01310fee1156febc950ee38605cd0711d283a52da67ad430) · [DID write](https://agung-testnet.subscan.io/tx/0x1a7acae3cb89c6c6c4f4f43fbd7d0c493e874c4f4b2f2d92d5efc8a438404e14)
- Contracts: [IdentityRegistry](https://agung-testnet.subscan.io/account/0x9E9463a65c7B74623b3b6Cdc39F71be7274e5971) · [MachineNFT](https://agung-testnet.subscan.io/account/0xB41C2A4f1c19b6B06beaAce0F5CD8439e77C4b1c) · [EventRegistry](https://agung-testnet.subscan.io/account/0x2DAD8905380993940e340C5cE6d313d5c2780040)

Activate does two txs: one mints the Machine NFT, one writes the DID.

## Undo

```bash
rm -rf /home/pi/peaqos
rm -rf /home/pi/.local/share/uv /home/pi/.cache/uv
```

Leaves the stock AutoPi install alone.