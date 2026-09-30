# MOS plugin architecture confirmed from mos-syncthing 0.2.0

The reference plugin confirms these important integration points:

- Backend commands are packaged to `/usr/bin/plugins/<plugin-name>`.
- Frontend files are packaged to `/var/www/mos-plugins/<plugin-name>/`.
- The frontend calls `/api/v1/mos/plugins/query`.
- `parse_json: true` lets the backend print JSON which MOS returns as `output`.
- Plugin settings use `/api/v1/mos/plugins/settings/<plugin-name>`.
- The Vue page is exposed through Vite Module Federation as `./Plugin`.
- Locales are exposed as `./Locales`.
- `manifest.json` is generated during the Vite build.
- The release workflow builds a Debian package and an MD5 file.

Disk Mover v0.2 follows those same conventions.
