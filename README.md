# pi-kit

Docker Sandboxes kit for the Pi coding agent ([pi.dev](https://pi.dev)), which
has no native `sbx` support. This kit defines it from scratch as `kind: sandbox`.

## 1. Install sbx

```bash
brew trust docker/tap
brew install docker/tap/sbx
sbx login
```

Last checked against sbx 0.42.1. The kit also runs on 0.38.0; see the auth
header note at the end of this file.

## 2. Build the image

The image pins the Node and `pi` versions, so the sandbox is reproducible
without a `setup.install` step re-running `npm install` on every create.

Docker Sandboxes pulls from a registry rather than your local store, so
without one you go through a tar file:

```bash
docker build --platform linux/arm64 -t pi-sandbox:0.85.1 .
docker image save pi-sandbox:0.85.1 -o pi-sandbox.tar
sbx template load pi-sandbox.tar
```

For a team, push to a registry and pin `sandbox.image` by digest instead:

```bash
docker build --platform linux/arm64 -t <your-org>/pi-sandbox:0.85.1 --push .
```

## 3. Register the OpenCode key

`opencode-go` authenticates with an API key (`OPENCODE_API_KEY`), not OAuth.
The key is already in `~/.pi/agent/auth.json` on the host:

```bash
python3 -c "import json,os;print(json.load(open(os.path.expanduser('~/.pi/agent/auth.json')))['opencode-go']['key'])" \
  | sbx secret set opencode-go
```

The secret name matches the `service:` id declared by the kit. On first run,
`sbx` asks you to approve the credential binding (required for third-party
kits on `schemaVersion: "2"`).

## 4. Set the network policy

Required once, before the first sandbox, and applies to all of them:

```bash
sbx policy init deny-all
```

The kit's own `permissions.network.allow` list layers on top, for its
sandboxes only.

## 5. Run

```bash
sbx kit validate ./
cd ~/your-project
sbx run --kit ~/path/to/pi-kit pi
```

## Model auth headers

`opencode-go` expects a different header depending on the model's API. The
kit injects `Bearer` by default:

| API | Models | Header |
| --- | --- | --- |
| `openai-completions` / `openai-responses` | `deepseek-v4-flash`, `deepseek-v4-pro`, `glm-5.1`, `glm-5.2`, `kimi-*`, `mimo-*`, `hy3`, `minimax-m2.7`, `qwen3.6-plus`, `gpt-5.6-luna`, `grok-4.5` | `Authorization: Bearer` |
| `anthropic-messages` | `minimax-m3`, `qwen3.7-max`, `qwen3.7-plus`, `qwen3.8-max` | `x-api-key` |

To use one of the four `anthropic-messages` models, uncomment the second
`inject` entry in `spec.yaml`.

The kit writes `header` and `format` out in full instead of using the
`scheme: bearer` shortcut. Both inject the key on sbx 0.42.1, but the
`x-api-key` row has no shortcut, so the explicit form keeps `spec.yaml` to a
single style. It is also the only form that works on sbx 0.38.0, where
`sbx kit validate` accepts `scheme` and the proxy then injects nothing,
leaving you with a `401`.
