# Syncing shared docs with other projects

Some projects (e.g. **stubhub-recon-gui**) include this documentation repo via a **local symlink**:

- **stubhub-recon-gui:** `docs/infrastructure-server-js` → points to this repo.  
  That path is in **.gitignore** (`docs/infrastructure-server-js/`), so the symlink is **never committed** in the other project—it’s local-only.

So the **only** place these docs are versioned is **this repo**. The other project’s repo never contains the symlink or its contents.

## How to keep in sync

1. **Edits via the symlink**  
   When you edit files under `docs/infrastructure-server-js/` from stubhub-recon-gui (or any project that symlinks here), you are editing **this repo’s** working tree. Those changes are **not** in the other project’s git at all (symlink is ignored).

2. **Commit and push from here**  
   After changing any of these docs (from this repo or via the symlink), run from **this repo**:
   ```bash
   cd /path/to/docs/infrastructure-server-js
   git status
   git add .
   git commit -m "Your message"
   git push
   ```

3. **Everyone else**  
   They need this repo cloned and the symlink created locally (e.g. `ln -s /path/to/docs/infrastructure-server-js docs/infrastructure-server-js`). Pulling this repo gives them the latest docs; the other project’s clone doesn’t contain the symlink.

## Current status

- **This repo:** Only place shared docs are committed. Commit and push from here to publish.
- **stubhub-recon-gui:** Symlink is **.gitignored**; docs under that path are never committed there. All shared doc changes must be committed in this repo.
