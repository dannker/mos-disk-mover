# Upgrade from v0.2 to v0.3

The v0.3 design now uses:

- Source disk + source folder.
- Destination disk + destination base.
- The complete relative source path is preserved under the destination base.
- Missing directories are reported during dry-run.
- Future COPY/MOVE will create only the missing directories.
- Dry-run uses rsync `-R` (`--relative`) and does not modify data.
- COPY and MOVE remain disabled.
