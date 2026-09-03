# bizmanage-skills

Shared, instance-agnostic Claude Code skills for working with [`bizmanage-cli`](https://github.com/bizmanagrcrm/bizmanage-cli) and BizManage CRM customization repos.

These apply to any BizManage instance (`https://<subdomain>.bizmanagecrm.com`) — nothing here is specific to one customer/tenant. Instance-specific facts (status IDs, custom table names, business rules) belong in that instance's own repo, not here.

## Skills

- **`skills/bizmanage-cli/`** — the `bizmanage` CLI: commands, options, config, troubleshooting.
- **`skills/bizmanage-customizations/`** — working inside a BizManage customization repo: file layout, backend scripts, reports, and building custom pages/actions (including the hidden-action + progress-bar bulk-operation pattern).

## Usage

Copy (or symlink) the `skills/*` directories you need into a project's `.claude/skills/` folder so Claude Code picks them up automatically, e.g.:

```bash
cp -r skills/bizmanage-cli skills/bizmanage-customizations /path/to/your-bizmanage-repo/.claude/skills/
```
