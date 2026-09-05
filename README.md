# Configure WinRM for Ansible

<p align="center">
  <img src="assets/hero.svg" alt="Configure WinRM for Ansible" width="100%">
</p>

A composite GitHub Action that configures PowerShell remoting (WinRM) on a Windows runner so Ansible can connect to it over `localhost`. It wraps [`ConfigureRemotingForAnsible.ps1`](https://github.com/ansible/ansible-documentation/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1), pinned to a commit SHA so the script can't change out from under a workflow.

Windows runners only. There's nothing to configure on Linux/macOS runners.

## Tutorial: add it to a workflow

Add a step that runs before your Ansible integration tests:

```yaml
jobs:
  integration:
    runs-on: windows-2025
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1

      - name: Configure WinRM for Ansible
        uses: lowlysre/config-ansible-remote-action@v1.0.0

      - name: Run ansible-test
        run: ansible-test integration
```

The action downloads and runs `ConfigureRemotingForAnsible.ps1` with `-Verbose -ForceNewSSLCert`, which enables the WinRM service, opens the firewall for it, and creates a self-signed HTTPS listener.

## How-to: pin a different script SHA

By default the action uses the SHA baked into `action.yml`. Pass `sha` to override it, for example to test a newer revision of the upstream script before this action's default is bumped:

```yaml
- name: Configure WinRM for Ansible
  uses: lowlysre/config-ansible-remote-action@v1.0.0
  with:
    sha: 1234567890abcdef1234567890abcdef12345678
```

Leave `sha` unset, or set it to `latest`, to use the pinned default.

## Reference

### Inputs

| Name  | Description                                                                                   | Required | Default    |
| ----- | ----------------------------------------------------------------------------------------------- | -------- | ---------- |
| `sha` | Commit SHA of `ansible/ansible-documentation` to pin `ConfigureRemotingForAnsible.ps1` to. `latest` uses the SHA bundled with this action. | No       | `latest`   |

### Outputs

None.

### Runner requirements

Windows only (`windows-latest`, `windows-2022`, `windows-2025`, etc). The action shells out with `pwsh`.

## Explanation: why pin a SHA at all

The upstream script lives on a `devel` branch with no version tags, so a bare link to it can change behavior underneath a workflow without warning. Pinning to a specific commit SHA keeps CI runs reproducible: the same `sha` input always downloads the exact same script. When ansible/ansible-documentation ships a new revision worth adopting, bump the default SHA in `action.yml` in a dedicated PR, rather than letting it drift silently.

## Explanation: why an action instead of copy-pasting the step

The inlined version of this step is four lines of PowerShell repeated in every workflow that needs it, which looks cheap until the SHA needs to move. At that point it's a grep-and-replace across every repo and workflow file that copied it, with no guarantee every copy is even in sync, versus bumping `action.yml` once here and letting consumers pick it up on their own schedule by moving their `@v1.0.0` pin.

It also collapses the diff a reviewer has to read. `uses: lowlysre/config-ansible-remote-action@v1.0.0` says what the step does; four lines of `ScriptBlock`/`WebClient` plumbing says how, and a reviewer unfamiliar with the Ansible remoting script has to go read it to know it's not doing anything else. Centralizing it here also means `.github/workflows/test.yml` is the one place this actually gets exercised, across every Windows runner image (`windows-2022`, `windows-2025`) and with the resulting WinRM listener checked, instead of every consumer's copy being an untested assumption that it still works.

## License

[MIT](LICENSE)
