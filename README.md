![bong — watch a pipeline. rip it. ding when done.](docs/hero.png)

# bong

watch a pipeline. rip it. ding when done.

```
$ bong
14:02:11  watching pipeline #987654 (root status: running)
14:02:11  #987654 running   https://gitlab.com/foo/bar/-/pipelines/987654
14:03:42  #987654 → discovered downstream #987655  https://gitlab.com/foo/deploy/-/pipelines/987655
14:03:42  #987655 created   https://gitlab.com/foo/deploy/-/pipelines/987655
14:04:11  #987655 manual    https://gitlab.com/foo/deploy/-/pipelines/987655
                ↑ desktop notification fires here ("needs manual trigger")
14:06:31  #987655 running
14:08:01  #987654 success
14:08:11  #987655 success
14:08:11  ✓ done — success: #987654 #987655
                ↑ second notification fires here
```

one bash file. zero deps beyond `glab`. mac + linux. caveman tier.

## install

prereq — install [`glab`](https://gitlab.com/gitlab-org/cli) and log in once:

```sh
glab auth login
```

then drop `bong` somewhere on your `$PATH`:

```sh
sudo install -m 0755 bong /usr/local/bin/bong
```

(or just `chmod +x bong && cp bong ~/bin/` — whatever. it's one script.)

## use

```sh
bong                                                  # latest pipeline on current branch (run from inside the repo)
bong 1234567                                          # by pipeline id (uses current repo for project context)
bong https://gitlab.com/foo/bar/-/pipelines/1234567   # by url — works from anywhere, including cross-project
```

what it does, every `BONG_POLL` seconds:

1. polls the pipeline you pointed it at
2. discovers any downstream / triggered child pipelines via the `bridges` api and adds them to the watch
3. fires a desktop notification + bell on **any** pipeline hitting `manual` (it'll keep watching after you trigger it)
4. exits when the whole tree is done. **`0`** if every pipeline ended `success`/`skipped`. **`1`** if any ended `failed`/`canceled`. final desktop notification + bell on the way out.

so you can chain:

```sh
bong && ./deploy.sh
```

ctrl-c to stop watching at any time.

## config

env vars (also see `.env.example`):

| var        | default | what                  |
|------------|---------|-----------------------|
| `BONG_POLL` | `10`    | seconds between polls |

`glab` itself handles auth / token / instance host — `bong` doesn't need any of that.

## notifications

best to worst, bong tries each in turn:

- **mac**: [`terminal-notifier`](https://github.com/julienXX/terminal-notifier) → `osascript` (built-in)
- **linux**: `notify-send` (libnotify; usually preinstalled with most desktops)
- **fallback**: terminal bell (`\a`)

if you're on a headless box, the bell is all you'll get — that's fine, exit code still works for `bong && deploy`.

## how it works (briefly)

- pipeline tree state lives in parallel bash arrays keyed by globally-unique pipeline id; we map id → project id so cross-project downstreams just work.
- json parsing is `grep`/`sed` — no `jq` dep. only the few fields we need (`id`, `project_id`, `status`, `web_url`).
- `manual` is treated as "stuck waiting on a human" — notify once, keep polling. only `success`/`failed`/`canceled`/`skipped` are terminal for exit.
- the script polls `glab api` directly rather than `glab ci ...` text output, since the json shape is more stable than the cli's pretty-printing.
