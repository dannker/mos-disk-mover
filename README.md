# MOS Disk Mover

Experimental MOS plugin to move or copy data directly between physical MOS pool members,
without transferring through the merged MergerFS mount.

This repository is structured after the current `ich777/mos-syncthing` MOS plugin layout.

## Current status — v0.2 scaffold

Safe/read-only functionality only:

- MOS plugin frontend structure using Vue + Vite Module Federation.
- MOS backend command under `/usr/bin/plugins/disk-mover`.
- Discover physical data-disk mount points below `/var/mergerfs/<POOL>/<DISK>`.
- Validate that source and destination are different physical member mount points.
- Read/save plugin settings.
- Poll backend status.

Actual COPY/MOVE operations are intentionally disabled.

## Planned milestones

- v0.3: rsync dry-run
- v0.4: real COPY
- v0.5: verified MOVE
- v0.6: background jobs, progress and cancel
- v1.0: SnapRAID-aware locking / operation checks

## Safety model

The backend, not the browser, will be responsible for validating every path.
The final mover must reject merged pool paths and arbitrary filesystem paths.
