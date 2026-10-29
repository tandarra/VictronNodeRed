# VictronNodeRed

Node-RED flow for the Tandarra Victron ESS controller (Pokeno).

- `flows.json` - the current full flow. Import into Node-RED (Menu > Import > Clipboard/File), replacing the existing tab.
- `CHANGELOG.md` - what changed in each version and why.

## Rollback
Every update is one commit. To get an older flow:

    git log --oneline -- flows.json
    git show <commit>:flows.json > flows-old.json

Tags (`v16.1`, `v16.3`, ...) mark each released version: `git show v16.1:flows.json`.
