# Persistent Windows VM for ServerRd

This is the architecture for a **single Windows instance reused across chained GitHub Actions runs**.

## What it preserves

With Azure VM hibernation enabled:

- installed applications
- application settings
- Windows registry
- files and project data
- Windows user profiles
- running application memory/state, when the VM successfully resumes from hibernation

Azure documents hibernation specifically for pre-warmed Windows VMs where applications can be initialized and then resumed in the desired state.

## Important distinction

The standard GitHub-hosted `windows-2025` runner is a new VM for each job. It cannot be turned into this persistent machine by workflow YAML alone.

The persistent machine is therefore:

**Azure Windows VM → GitHub self-hosted runner → RDP/Tailscale**

GitHub Actions only controls the VM lifecycle.

## 1. Create the VM

Use a Windows configuration supported by Azure hibernation. Azure currently documents Windows 10 Pro/Enterprise, Windows 11 Pro/Enterprise, and supported Windows Server versions, with supported general-purpose VM sizes up to 64 GB RAM.

Example:

```bash
az vm create \
  --resource-group ServerRdRG \
  --name ServerRdVM \
  --image Win2022AzureEdition \
  --size Standard_D2s_v5 \
  --admin-username azureuser \
  --public-ip-sku Standard \
  --enable-hibernation true \
  --nsg-rule RDP
```

Do not use an ephemeral OS disk.

## 2. Install the GitHub Actions runner inside that VM

On the Windows VM:

1. Open the repository's GitHub Settings.
2. Go to **Actions → Runners → New self-hosted runner**.
3. Select Windows x64.
4. Download/extract the runner.
5. Configure it with labels:

```
self-hosted
windows
serverRd
```

6. Install it as a Windows service so it starts automatically after resume/reboot.

The workflow in `.github/workflows/persistent-windows.yml` targets exactly those labels.

## 3. Configure GitHub secrets

Repository → Settings → Secrets and variables → Actions → Secrets:

```
AZURE_CREDENTIALS
TS_AUTHKEY
```

`AZURE_CREDENTIALS` is the JSON credential used by `azure/login@v2`.

The Azure identity needs permission to start/deallocate the specific VM.

## 4. Configure GitHub variables

Repository → Settings → Secrets and variables → Actions → Variables:

```
AZURE_RESOURCE_GROUP=ServerRdRG
AZURE_VM_NAME=ServerRdVM
RDP_DAYS=7
```

## 5. First run

Run:

**Actions → ServerRd Persistent Windows Session → Run workflow**

The controller:

1. starts/resumes the Azure VM
2. waits for the self-hosted runner
3. runs the interactive Windows session on that exact VM
4. keeps it alive for the session window
5. hibernates the VM
6. dispatches the next workflow automatically

## 6. Next run

The next workflow first starts the same Azure VM.

If Azure reports that the VM resumes from hibernation, Windows restores the saved memory state. The GitHub runner reconnects and the interactive job runs on the same machine.

If the VM instead performs a cold boot, disk state still persists, but live application RAM state was not restored. The workflow logs make this distinction visible.

## 7. Critical rule

Do **not** put `powercfg /h off` into the persistent workflow.

That command disables Windows hibernation and defeats the entire purpose of this architecture.

The old `vm-test.yml` is still available for the original GitHub-hosted runner mode. The new persistent workflow is separate so the existing setup is not destroyed.
