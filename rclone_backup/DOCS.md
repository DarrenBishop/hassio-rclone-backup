# Rclone Backup

Backup your Home Assistant configuration or backups to over [40 cloud providers](https://rclone.org/#providers) using [Rclone](https://rclone.org/).

```yaml
jobs:
  - name: Sync Daily Backups
    schedule: 10 4 * * *
    command: sync
    sources:
      - /backup
    destination: 'google:/Backup/Home Assistant'
    include:
      - DailyBackup*
    exclude: []
    flags: {}
dry_run: false
config_path: /homeassistant/rclone.conf
```

## Configuration

**Option:** `jobs`

A list of jobs to schedule, see the [job config](#job-config) section for a list of available options.

**Option:** `flags`

Global map of flags to give the rclone command for every job, see [rclone flags](https://rclone.org/flags). Underscores are allowed for the flags and they will be replaced with dashes.

```yaml
jobs:
  - ...
flags:
  drive-use-trash: false
  min-age: 30d
```

*For example, you may want to add `drive-use-trash: false` when using google drive so rclone immediately deletes files instead of sending them to the trash.*

**Option:** `extra_flags`

List of flags to give the rclone command, applied globally to all jobs. For use when `flags` option isn't working, the list of flags are appended directly to the rclone command.

```yaml
jobs:
  - ...
extra_flags:
  - --drive-use-trash=false
  - --min-age=30d
```

**Option:** `dry_run`

Trial run with no permanent changes, see what rclone would do without actually doing it.

**Option:** `run_once`

Will run all jobs with no schedule defined immediately then exit. Designed for use with the `hassio.addon_restart` service to trigger jobs.

**Option:** `config_path`

The location of the rclone config file, must be stored under `/config`.

**Option:** `rclone_config`

This allows you to specify the rclone config from within the addon configuration, defining it will take precedence over the `config_path` option.

```yaml
jobs:
  - ...
rclone_config: |
  [google]
  type = drive
  scope = drive
  token = REDACTED
  use_trash = false
```

**Option:** `no_rename`

Disable the renaming of backups before uploading them, with this enabled backups will use their slug id (their name on disk) instead of their friendly name.

**Option:** `no_unrename`

Prevents renaming backups back to their original name, with this enabled the backup files will be renamed to use their friendly name and this will persist on disk.

*Note: this option can apparently cause issues with restoring backups through the WebUI, however, selecting **⋮** → **Reload** should fix it. 
This won't affect the actual integrity of the backups.*

**Option:** `no_slugify`

This will disable the slugification of backup names. This means the **user** is responsible for ensuring their backup names are **valid** filenames for their filesystem and the destination filesystem.

**Option:** `no_events`

Disable sending completion and failure events to Home Assistant.

**Option:** `remote_control`

Starts a **dedicated, authenticated** rclone [remote-control (rc) API](https://rclone.org/rc/) daemon (on port `5572`) so external clients — such as the [CloneAssistant](https://github.com/nielash/cloneassistant) integration — can start, stop and monitor rclone jobs over the network. See [Using with CloneAssistant](#using-with-cloneassistant).

This is a separate process from the Web UI daemon, so **the Web UI keeps its password-less ingress login untouched**. When disabled (the default) no extra process runs. When enabled you **must** set `rc_username` and `rc_password`.

**Option:** `rc_username` / `rc_password`

The username and password required to authenticate against the rc API daemon when `remote_control` is enabled. Enter the same values in the CloneAssistant integration. (These do **not** affect the Web UI.)

**Option:** `rc_job_expire`

How long rclone keeps finished async job results so a client can poll them, see [`--rc-job-expire-duration`](https://rclone.org/rc/#job-expiry). Defaults to `1h`. CloneAssistant requires at least `1h` (rclone's own default of `60s` is too short); only change this if you know you need to.

**Option:** `rc_allow_origin`

Advanced/optional. Sets the allowed CORS origin for the rc API daemon (`--rc-allow-origin`). Leave empty unless you access the rc API from a browser on a different origin. CloneAssistant is a server-side client and does **not** need this.

## Job Config

**Option:** `sources`

List of source locations in the format `remote:path` or for local paths just `path`, e.g. `/backup`, `/config`, `/share`, `/ssl`, `/media`.

**Option:** `source`

A slightly cleaner way to specify a single source, this option will override the `sources` option.

**Option:** `destinations`

A list of locations to write to in the format `remote:path` or for local paths just `path`, see [rclone docs](https://rclone.org/docs).

**Option:** `destination`

A slightly cleaner way to specify a single destination, this option will override the `destinations` option.

> **Multiple Sources**
>
> When there are multiple sources each source will be backed-up to the destination under `destination/source`
> otherwise they will be directly under `destination`.
>
> **Multiple Destinations**
>
> All sources are backed-up to each destination following the rules for multiple sources mentioned above.

**Option:** `name`

Optionally you can provide a friendly name for the job, this can be useful to identify which job is being run when you have multiple.

**Option:** `schedule`

Specify when the rclone backup should run using cron syntax. If the `schedule` option is empty or undefined the job will be run when the addon starts.

**Option:** `command`

The rclone command to run e.g. `sync`, `copy`, `move`.

**Option:** `include`

List of files or folders to include, see [rclone filtering](https://rclone.org/filtering).

*Note: these filters are matching the backups name on the filesystem, if you have not disabled renaming then this will be a slugified version of the backups name, this will effectively replace all special characters with an `_` and remove duplicate underscores.*

To preview the slugified names of your backups you can add a startup job like the following.

```yaml
jobs:
  - command: ls
    name: List Local Backups
    sources:
      - /backup
```

**Option:** `exclude`

List of files or folders to exclude, see [rclone filtering](https://rclone.org/filtering).

**Option:** `flags`

Map of flags to give to the rclone command, see [rclone flags](https://rclone.org/flags).

**Option:** `extra_flags`

List of flags to give the rclone command, applied globally to all jobs. For use when `flags` option isn't working, the list of flags are appended directly to the rclone command.

---

### Configuring Rclone Remotes

The addon now supports ingress and the Rclone Web UI, you can access this by clicking the **Open Web UI** button in the addon info panel. You do not need a username or password and can just click the login button. Then you can click **Configs** -> **Create new config** to create a new remote.

For more information, or to configure rclone without using the WebUI, you can consult the [rclone docs](https://rclone.org/docs/). The rclone config can be found at `/config/rclone.conf` by default.

---

## Using with CloneAssistant

[CloneAssistant](https://github.com/nielash/cloneassistant) is a Home Assistant integration that lets you *control and monitor* recurring rclone jobs (`sync`, `bisync`, `check`, …) from dashboards — each job becomes a device with a switch, run-stat sensors and health/last-success sensors. Unlike this addon's scheduler (which shells out to the `rclone` binary on a cron), CloneAssistant is purely a **client of the rclone rc API**: it needs a running, authenticated `rclone rcd` it can reach over the network.

Enabling `remote_control` starts a **dedicated rc API daemon on port `5572`**, authenticated with `rc_username`/`rc_password`. It runs alongside — but separately from — the daemon that serves the Web UI, so **the Web UI keeps its existing password-less ingress login**; only the network-exposed API requires credentials. The daemon shares the same rclone config and remotes.

> **Why a network port needs a password:** the rc API can run `sync`/`delete`/`move` against every configured remote. Once it's reachable on a port (rather than ingress, which Home Assistant gates), an unauthenticated daemon is effectively remote control of your storage for anyone on the network — so CloneAssistant (rightly) requires credentials.

### Setup

1. **Enable the rc API.** In the addon **Configuration** tab set:

   ```yaml
   remote_control: true
   rc_username: choose-a-username
   rc_password: choose-a-strong-password
   # rc_job_expire: 1h   # optional, this is the default
   ```

2. **Expose the rc port.** In the addon **Network** tab, assign a host port to `5572/tcp` (e.g. `5572`). This is what makes the API reachable from Home Assistant / other LAN machines.

3. **Restart the addon.** The log should report *"Starting Rclone rc API daemon on :5572 (authenticated)"*.

4. **Add the integration in CloneAssistant** using:
   - **Host:** `homeassistant.local:5572` (or `<your-HA-ip>:<the host port you assigned>`). Prefix with `https://` only if you put rclone behind a trusted TLS proxy — self-signed certs are not supported.
   - **Username / Password:** the `rc_username` / `rc_password` you set above.
   - **command / command_args:** the rc method and arguments for the job, e.g. command `sync/sync` with args `{ "srcFs": "/backup", "dstFs": "google:/Backup/Home Assistant" }`.

> **Security**
>
> Exposing a host port publishes the rc API to your network — anyone who can reach that port **and** has the credentials can drive rclone against your remotes. Only assign the host port once `rc_username`/`rc_password` are set, keep the credentials strong, and don't forward the port to the internet. For access across machines, [Tailscale](https://tailscale.com/) (as CloneAssistant's docs recommend) is safer than port-forwarding.

---

## Events

> Blueprint to send notifications based on these events. [[source]](https://github.com/jcwillox/home-assistant-blueprints)
>
> [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A//raw.githubusercontent.com/jcwillox/home-assistant-blueprints/main/automation/notify_rclone_backup.yaml)

The addon will fire events using Home Assistant's event bus whenever a job is completed or fails.

**Event:** `rclone_backup.job_successful`

**Event:** `rclone_backup.job_failed`

Both events will have the following attributes.

| Attribute     | Description                                            |
| ------------- | ------------------------------------------------------ |
| `name`        | The name of the job.                                   |
| `command`     | The rclone command that was run.                       |
| `source`      | The source location.                                   |
| `destination` | The destination location. (optional)                   |
| `error`       | The error message if the job failed. (optional)        |
| `duration`    | The duration of the job as a human string, eg. `1m2s`. |
| `seconds`     | The duration of the job in seconds.                    |
