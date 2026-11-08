blog_post
=====

Companion repo to the 2016 blog post
[Rebar3 Example Release](https://markusfeyh.com/posts/rebar3-example-release/).
It shows a rebar3 release that pulls in hex dependencies, and that is configured
per environment with profiles (`dev`, `stage`, `prod`), relx `overlay_vars` and a
templated `sys.config`.

Tested with Erlang/OTP 29.1.2 and rebar3 3.28.0. Building jiffy's NIF also needs
a C compiler.

Build
-----

    $ rebar3 compile

Releases per environment
------------------------

`etc/sys.config` is a template:

    [
      { blog_post, [{dynamic_env, {{dynamic_var_value}}}]}
    ].

Each profile points `overlay_vars` at a different file in `config/`, and the
`template` overlay renders `etc/sys.config` into the release with that value:

| Command                    | overlay_vars file       | `dynamic_env` in the release |
|----------------------------|-------------------------|------------------------------|
| `rebar3 release`           | `config/default.config` | `default`                    |
| `rebar3 as dev release`    | `config/dev.config`     | `dev`                        |
| `rebar3 as stage release`  | `config/stage.config`   | `stage`                      |
| `rebar3 as prod release`   | `config/prod.config`    | `prod`                       |

For example:

    $ rebar3 as prod release
    $ cat _build/prod/rel/blog_post/releases/0.1.0/sys.config
    [
      { blog_post, [{dynamic_env, prod}]}
    ].
    $ _build/prod/rel/blog_post/bin/blog_post daemon
    $ _build/prod/rel/blog_post/bin/blog_post eval "application:get_env(blog_post, dynamic_env)."
    {ok, prod}
    $ _build/prod/rel/blog_post/bin/blog_post stop

The `dev`, `stage` and `prod` profiles also set `{dev_mode, false}` and
`{include_erts, true}`, so they build self-contained releases.

What changed since the 2016 blog post
-------------------------------------

The profiles, `overlay_vars` and `sys.config` template work exactly as the post
describes. The dependency section has aged:

- **No overrides needed any more.** jiffy (now 2.0.2) builds its NIF with
  rebar3 by itself, so the `pc` override for jiffy is gone. The riakc overrides
  fetched plugins over `git://github.com/...` URLs, which GitHub turned off in
  2022. The old `rebar.config` also defined `overrides` twice, and rebar3 only
  uses one of them, so the riakc overrides never actually took effect.
- **riakc comes from OpenRiak instead of hex.** The newest riakc on hex (2.5.3,
  from 2017) doesn't compile on OTP 27+: `maybe` is now a keyword, and it uses
  the removed `queue()` type. Riak and its Erlang client are now maintained by
  [OpenRiak](https://github.com/OpenRiak) (with the Erlang Ecosystem
  Foundation), which doesn't publish to hex. So riakc is a git dependency on
  the `openriak-3.4` branch, pinned to an exact commit by `rebar.lock`.
- **The release now includes its dependencies.** jiffy and riakc are listed in
  `blog_post.app.src`, so relx puts them in the release.
- **`etc/vm.args` is used again** (`{vm_args, "etc/vm.args"}`), and the
  `+K true` flag is gone. It has done nothing since OTP 21.

Current relx can also substitute environment variables when the release
*starts*, which `overlay_vars` can't: use `{sys_config_src, "config/sys.config.src"}`
and `{vm_args_src, "config/vm.args.src"}`, and write `${VAR}` in those files.
That way one release artifact can serve every environment, instead of building
one release per profile.
