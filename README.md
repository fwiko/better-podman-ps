# ./pops - better `podman ps` 
A replacement for the default podman-ps that tries really hard to fit within
your terminal width.

> [!NOTE]
> This is a fork of
> [Mikescher/better-docker-ps](https://github.com/Mikescher/better-docker-ps),
> migrated to provide the same functionality for `podman ps` instead of
> `docker ps`.

> [!WARNING]
> Mac OS support has been removed as I do not have the ability to
> confidently test.

![](readme.d/main.png)

## Rationale

By default, my `podman ps` output is really wide and every line wraps around
into three. This (obviously) breaks the tabular display and makes everything  
chaotic. *(This gets especially bad if one container has multiple port mappings,
and they are all displayed in a single row)* It doesn't look like we'll get
improved output in the foreseeable future (see
[moby#7477](https://github.com/moby/moby/issues/7477)), so I decided to make my
own drop-in replacement.  

## Features

 - All normal commandline flags/options from podman-ps work *(almost)* the same.
 - Write multi-value data (like multiple port mappings, multiple networks, etc.)
   into multiple lines instead of concatenating them.
 - Add color to the STATE and STATUS column (green / yellow / red).
 - Automatically remove columns in the output until it fits in the current
   terminal width.
 - sort the output with the `--sort` argument
 - Enter watch mode with the `--watch` argument

More Changes from default podman-ps:
 - Show (by default) the container-cmd without arguments.
 - Show the ImageName (by default) without the registry prefix, and split
   ImageName and ImageTag into two columns.
 - Added the columns IP and NETWORK to the default column set (if they fit)
 - Added support for a few new columns (via --format):  
   `{{.ImageName}`, `{{.ImageTag}`, `{{.Tag}`, `{{.ImageRegistry}`,
   `{{.Registry}`, `{{.ShortCommand}`, `{{.LabelKeys}`, `{{.IP}`, `{{.User}`  
 - The `{{.User}}` column shows the user a container runs as (`Config.User`).
   Because this is not part of the cheaper container-list endpoint, pops only
   queries the (slower) container-inspect endpoint when this column is used.
 - Added options to control the color-output, the used socket, the time-zone and
   time-format, etc (see `./pops --help`)

## Getting started

### Generic Linux (e.g. Debian/Fedora/...)
 - Download the latest binary from the
   [releases page](https://github.com/fwiko/better-podman-ps/releases) and put
   it into your PATH (eg /usr/local/bin)
 - You can also use the following one-liner (afterwards you can use the `pops`
   command everywhere):
```
sudo wget "https://github.com/fwiko/better-podman-ps/releases/latest/download/pops_linux-amd64-static" -O "/usr/local/bin/pops" && sudo chmod +x "/usr/local/bin/pops"
```

### ArchLinux
 - Alternatively you can use one of the AUR packages (under Arch Linux):
    * https://aur.archlinux.org/packages/pops-bin (installs `pops` into your
      PATH)
    * https://aur.archlinux.org/packages/pops-git (installs `pops` into your
      PATH)
 - or the homebrew package:
    * `brew tap mikescher/tap && brew install pops`

### Optional steps
 - Alias the podman ps command to `pops` (see
   [section below](#usage-as-drop-in-replacement))

### Building from source

If you want to build `pops` from source, you need to have Go installed.

```sh
git clone https://github.com/fwiko/better-podman-ps.git
cd better-podman-ps
make build
mv _out/pops "$HOME/.local/bin/"
```

## Screenshots

![](readme.d/fullsize.png)  
All (default) columns visible

&nbsp;

![](readme.d/default.png)  
Output on a medium sized terminal

&nbsp;

![](readme.d/small.png)  
Output on a small terminal

&nbsp;

## Usage as drop-in replacement

You can fully replace podman ps by creating a shell function in your `.bashrc` /
`.zshrc`...

~~~sh
podman() { 
  case $1 in 
    ps) 
      shift 
      command pops "$@" 
      ;; 
    *) 
      command podman "$@";; 
  esac 
} 
~~~

This will alias every call to `podman ps ...` with `pops ...` (be sure to have
the pops binary in your PATH).

If you are using the fish-shell you have to create a (similar) function:

~~~fish
function podman
    if test -n "$argv[1]"
        switch $argv[1]
            case ps
                pops $argv[2..-1]
            case '*'
                command podman $argv[1..-1]
        end
    end
end
~~~

## Changing the output format

By default pops tries to be "intelligent" and find the best output format for
your terminal width. The current output formats (= table columns) are defined in
the
[options.go](https://github.com/fwiko/better-podman-ps/blob/master/cli/options.go).
The first format that fits in your terminal width is used.

But you can also override it by supplying a `--format` parameter. If you supply
more than one `--format` parameter the first one that fits your terminal is used
(same logic as with the default ones...)

Normally only simple columns aka `{{.Status}}` are supported.  
But you can also use the full golang template syntax (e.g.
`{{ printf "%.15s" .Command }}`). In this case it can be useful to specify the
column header by prefixing it with a colon
(`SHORTENED NAME:{{ printf "%.10s" (join .Names ";") }}`)

The following functions are defined in these templates (plus the
[default go functions](https://pkg.go.dev/text/template)):
 - `join`: strings.Join
 - `array_last`: v\[-1\]
 - `array_slice`: v\[a..b\]
 - `in_array`: v1.contains(v2)
 - `json`: json.Marshal(v)
 - `json_indent`: json.MarshalIndent(v, "", " ")
 - `json_pretty`: json.Indent(v, "", " ")
 - `coalesce`: v1 ?? v2
 - `to_string`: fmt.Sprintf("%v", v)
 - `deref`: *v
 - `now`: time.Now()
 - `uniqid`: UUID

Examples:
~~~~
$ ./pops --format "table {{.ID}}"
$ ./pops --format "table {{.ID}}\\t{{.Names}}\\t{{.State}}"

$ ./pops --format "idlist"

$ ./pops --format "table {{.ID}}\\t{{.Names}}\\t{{.State}}"  --format "table {{.ID}}\\t{{.Names}}" --format "table {{.ID}}"

$ ./pops --format "ID: {{.ID}}; Name: {{.Names}}"

$ ./pops -aq

$ ./pops --sort "IP" --sort-direction "ASC"

$ ./pops --format "table {{.ID}}\\tCMD:{{ printf \"%.15s\" .Command }}"
$ ./pops --format "table {{.ID}}\\tNAME:{{ printf \"%.10s\" (join .Names \";\") }}"

~~~~

## Persistant configuration

You can also configure some/most of the options via a configuration file.  
Place a TOML formatted file in `$HOME/.config/pops.conf` /  
`$XDG_CONFIG_HOME/pops.conf`. ( `~/Library/Application Support/pops.conf` is
also supported under macOS )

The first existing file from the following locations is used:
 - `$XDG_CONFIG_HOME/pops.conf` (or the platform default, e.g.
   `~/Library/Application Support/pops.conf` under macOS)
 - `~/.config/pops.conf`

The following keys are supported:
 - verbose
 - silent
 - timezone
 - timeformat
 - timeformat-header
 - color
 - socket
 - all
 - size
 - filter (= string array)
 - search
 - format (= string array)
 - last
 - latest
 - truncate
 - header (= true / false / simple)
 - sort (= string array)
 - sort-direction (= string array)

Example:
```toml
verbose = 0

timezone = "Europe/Berlin"

format = [
   "table {{.ID}}\t{{.Names}}\t{{.State}}\t{{.Status}}",
   "table {{.ID}}\t{{.Names}}\t{{.State}}",
   "table {{.ID}}\t{{.Names}}",
   "table {{.ID}}",
]

header = "simple"
```

## Manual

Output of `./pops --help`:

~~~~~~
better-podman-ps

Usage:
  pops [OPTIONS]                     List podman container

Options (default):
  -h, --help                         Show this screen.
  --version                          Show version.
  --all , -a                         Show all containers (default shows just running)
  --filter <ftr>, -f <ftr>           Filter output based on conditions provided
  --search <str>, -g <str>           Filter output by substring match across all visible columns (case-insensitive)
  --format <fmt>                     Pretty-print containers using a Go template
  --last , -n                        Show n last created containers (includes all states)
  --latest , -l                      Show the latest created container (includes all states)
  --no-trunc                         Don't truncate output (eg ContainerIDs, Sha256 Image references, commandline)
  --quiet , -q                       Only display container IDs
  --size , -s                        Display total file sizes

Options (extra | do not exist in `podman ps`):
  --silent                           Do not print any output
  --timezone                         Specify the timezone for date outputs
  --color <true|false>               Enable/Disable terminal color output
  --no-color                         Disable terminal color output
  --socket <filepath>                Specify the podman socket location (Default: `auto`
  --timeformat <go-time-fmt>         Specify the datetime output format (golang syntax)
  --no-header                        Do not print the table header
  --simple-header                    Do not print the lines under the header
  --format <fmt>                     You can specify multiple formats and the first one that fits your terminal widt will be used
  --sort <col>                       Sort output by a specific column, use the same identifier as in --format, only useful together with table formats 
  --sort-direction <ASC|DESC>        The sort direction, only useful in combination with --sort
  --watch <interval>                 Automatically refresh output periodically (interval is optional, default: 2s)

Available --format keys (default):
  {{.ID}}                            Container ID
  {{.Image}}                         Image ID
  {{.Command}}                       Quoted command
  {{.CreatedAt}}                     Time when the container was created.
  {{.RunningFor}}                    Elapsed time since the container was started.
  {{.Ports}}                         Published ports. ([!] differs from podman CLI, these are only the published ports)
  {{.State}}                         Container status
  {{.Status}}                        Container status with details
  {{.Size}}                          Container disk size.
  {{.Names}}                         Container names.
  {{.Labels}}                        All labels assigned to the container.
  {{.Label}}                         [!] Unsupported
  {{.Mounts}}                        Names of the volumes mounted in this container.
  {{.Networks}}                      Names of the networks attached to this container.

Available --format keys (extra | do not exist in `podman ps`):
  {{.ImageName}                      Image ID (without tag and registry)
  {{.ImageTag}, {{.Tag}              Image Tag
  {{.ImageRegistry}, {{.Registry}    Image Registry
  {{.ShortCommand}                   Command without arguments
  {{.LabelKeys}                      All labels assigned to the container (keys only)
  {{.ShortPublishedPorts}}           Published ports, shorter output than {{.Ports}}
  {{.LongPublishedPorts}}            Published ports, full output with IP
  {{.ExposedPorts}}                  Exposed ports
  {{.NotPublishedPorts}}             Exposed but not published ports
  {{.PublishedPorts}}                Published ports
  {{.PublicPorts}}                   Only the public part of published ports
  {{.IP}                             Internal IP Address
  {{.User}                           User the container runs as (queried via container-inspect)
~~~~~~
