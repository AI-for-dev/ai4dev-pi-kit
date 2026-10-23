# ai4dev-pi-kit

Docker Sandboxes kit for the Pi coding agent ([pi.dev](https://pi.dev)), which
is not one of the agents `sbx` runs out of the box. The kit is a
`kind: sandbox` that boots Docker's published Pi image and adds the provider
credentials, the network allowlist and the Pi configuration used in the course.

Docker publishes a different kit, `docker.io/sbx/pi-kit`, which only wires
Anthropic credentials. [pi-sandbox](https://github.com/carderne/pi-sandbox) is
unrelated too: it is a Pi extension that wraps bash commands with
`bwrap` or `sandbox-exec` on the host.

```
ai4dev-pi-kit
├── spec.yaml
└── files/home/.pi/agent/settings.json
```

## 1. Install sbx

Follow https://docs.docker.com/ai/sandboxes/install/, then:

```bash
sbx login
```

Last checked against sbx 0.45.1. The kit needs 0.42.1 or later: older
releases accept `scheme: bearer` in `sbx kit validate` but the proxy then
injects nothing, and every model call fails with a `401`.

## 2. The image

`spec.yaml` boots `docker.io/sbx/pi-image:latest`, which Docker builds from
[docker/sbx-kits-contrib](https://github.com/docker/sbx-kits-contrib/tree/main/pi):
the `shell-docker` sandbox template (Node 22.22.1, git, ripgrep, python3, uv)
plus `fd` and Pi. Docker rebuilds it every night against Pi's latest npm
release, so there is nothing to build here and `sbx` pulls it from Docker Hub
on the first run.

`latest` therefore moves: two sandboxes created a week apart can run two
different Pi versions. When runs must be comparable, as in a measurement
campaign, pin the image by digest in `spec.yaml`:

```bash
docker buildx imagetools inspect docker.io/sbx/pi-image:latest   # prints the digest
```

```yaml
sandbox:
  image: docker.io/sbx/pi-image@sha256:<digest>
```

## 3. The course models

`settings.json` declares one Pi package:

```json
"packages": ["npm:@ai-for-dev/providers"]
```

Pi installs it from npm the first time it starts.
[@ai-for-dev/providers](https://github.com/AI-for-dev/providers) registers the
two providers used in the course:

| Provider | Server | Key |
| --- | --- | --- |
| `ia4dev-ilaas` | `https://devllm.ilaas.fr/v1` | `ILAAS_API_KEY` |
| `ia4dev-gricad` | `https://ia4dev-owui.univ-grenoble-alpes.fr/api` | `GRICAD_API_KEY` |

The package asks each server for its model list at startup, when you open
`/model` and after `/login`. It also holds the context window, the vision flag
and the reasoning settings measured for each model. A model the package does
not know stays usable and takes the context window the server reports.

`npm:@ai-for-dev/providers` resolves to whatever npm serves the first time a
sandbox runs it, so two sandboxes a release apart can see different models.
Pin the version to make them comparable:

```json
"packages": ["npm:@ai-for-dev/providers@0.4.1"]
```

## 4. Register the provider keys

Each secret name matches a `service:` id declared by the kit:

```bash
sbx secret set ilaas      # provider ia4dev-ilaas
sbx secret set gricad     # provider ia4dev-gricad
```

Only `ilaas` is `required: true`. A sandbox starts with the ILAAS key alone,
and `ia4dev-gricad` then lists no model.

`opencode-go` authenticates with an API key (`OPENCODE_API_KEY`), not OAuth.
If the key is already in `~/.pi/agent/auth.json` on the host:

```bash
python3 -c "import json,os;print(json.load(open(os.path.expanduser('~/.pi/agent/auth.json')))['opencode-go']['key'])" \
  | sbx secret set opencode-go
```

On the first interactive run, `sbx` asks you to approve the credential binding
(required for third-party kits on `schemaVersion: "2"`). `sbx create` and
scripts get no prompt: the sandbox starts anyway, the proxy never swaps the
sentinel, and the first model call fails with a `401`. Write the bindings in
`~/.config/sbx/credentials.yaml` beforehand:

```yaml
bindings:
  ilaas:
    apiKey:
      domains: [devllm.ilaas.fr]
  gricad:
    apiKey:
      domains: [ia4dev-owui.univ-grenoble-alpes.fr]
  opencode-go:
    apiKey:
      domains: [opencode.ai]
```

## 5. Set the network policy

Required once, before the first sandbox, and applies to all of them:

```bash
sbx policy init deny-all
```

The kit's own `permissions.network.allow` list layers on top, for its
sandboxes only. It must include every domain a credential is injected into:
`sbx` does not allow those implicitly, so without `devllm.ilaas.fr`,
`ia4dev-owui.univ-grenoble-alpes.fr` and `opencode.ai` in the list every model
call is refused. `registry.npmjs.org` is where Pi fetches
`@ai-for-dev/providers`, and what `pi install npm:...` needs afterwards.

## 6. Run

```bash
sbx kit validate ~/path/to/ai4dev-pi-kit
cd ~/your-project
sbx run ~/path/to/ai4dev-pi-kit
```

## Model auth headers

`opencode-go` expects a different header depending on the model's API. The
kit injects `Authorization: Bearer` (`scheme: bearer`):

| API | Models | Header |
| --- | --- | --- |
| `openai-completions` / `openai-responses` | `deepseek-v4-flash`, `deepseek-v4-pro`, `glm-5.1`, `glm-5.2`, `kimi-*`, `mimo-*`, `hy3`, `minimax-m2.7`, `qwen3.6-plus`, `gpt-5.6-luna`, `grok-4.5` | `Authorization: Bearer` |
| `anthropic-messages` | `minimax-m3`, `qwen3.7-max`, `qwen3.7-plus`, `qwen3.8-max` | `x-api-key` |

`scheme` has no shortcut for `x-api-key`. To use one of the four
`anthropic-messages` models, replace the `opencode.ai` inject entry in
`spec.yaml` with:

```yaml
        - domain: opencode.ai
          header: x-api-key
          format: "%s"
```

Both course providers use `openai-completions` and read
`Authorization: Bearer`, so `scheme: bearer` is right for them too.
