# osgar-datahub (on Stella)

Dev / staging .NET app at `osgar.datahub.foxcraft.digital`. Host-wide .NET pattern: [`../../dotnet-app-deployment.md`](../../dotnet-app-deployment.md). Per-app SSH: [`../../dev-ssh-access.md`](../../dev-ssh-access.md).

| Document | Contents |
|----------|----------|
| [setup.md](setup.md) | Permissions (ACL), Node, supervisord, SCSS watcher |
| [ssh-sqlcmd.md](ssh-sqlcmd.md) | `sqlcmd` / mssql-tools18 inside the SSH container |
| [ssh-app-logs.md](ssh-app-logs.md) | Shared `app-logs` mount for SSH log access |

Edison mounts this tree via sshfs as user `edison`: [`../../../edison/cursor-agent.md`](../../../edison/cursor-agent.md).
