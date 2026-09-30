# MOS Disk Mover

MOS plugin for safely moving or copying data directly between physical MOS pool members,
without operating through the merged MergerFS mount.

## v0.3 — Destination base + preserved source structure + dry-run

Implemented:

- Native MOS Vue plugin page.
- Physical pool-member discovery under `/var/mergerfs/<POOL>/<DISK>`.
- Safe folder browser constrained to the selected physical disk.
- Source folder selection (`/` means the complete source disk).
- Destination **base** selection.
- The source folder path is always preserved below the destination base.
- Dry-run reports the final resolved destination and which directories would need creation.
- Path traversal / symlink escape protections.
- `rsync -n -R` dry-run transfer simulation.
- File counts, bytes to transfer, destination free space and space check.
- COPY and MOVE remain intentionally disabled.

## Example

```text
Source disk:       MEDIA / disk1
Source folder:     /Movies/4K/Marvel

Destination disk:  MEDIA / disk3
Destination base:  /
```

Resolved destination:

```text
/var/mergerfs/MEDIA/disk3/Movies/4K/Marvel
```

If only `/Movies` already exists on `disk3`, a future real transfer would create only:

```text
/Movies/4K
/Movies/4K/Marvel
```

Another example:

```text
Destination base: /Archive
```

resolves to:

```text
/var/mergerfs/MEDIA/disk3/Archive/Movies/4K/Marvel
```

v0.3 only simulates this behavior and never creates directories or writes data.

## Backend commands

```text
disk-mover disks
disk-mover folders DISK [RELATIVE_FOLDER]
disk-mover validate SOURCE_DISK SOURCE_FOLDER DEST_DISK DESTINATION_BASE
disk-mover dry-run SOURCE_DISK SOURCE_FOLDER DEST_DISK DESTINATION_BASE [PRESERVE_XATTRS=1] [PRESERVE_ACL=1]
disk-mover status
```

## Safety

COPY and MOVE remain blocked in both the UI and backend.

## Roadmap

- v0.4: real COPY with collision safeguards and automatic directory creation.
- v0.5: verified MOVE.
- v0.6: background jobs, progress and cancel.
- v1.0: SnapRAID-aware locking and operation checks.
