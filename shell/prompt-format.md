# Prompt Format

Spending time at a shell prompt means having an opinion about interactive shell (fish, another topic)
and prompt format.

My initial approach was a [custom fish theme][1] leveraging [robertgzr/porcelain][2].

But I've recently discovered [starship/starship][3] and I'll admit that the combination of its speed
(implemented in Rust) and flexibiilty (plenty of [configuration options][4]) is compelling.

I've tweaked things, of course, and I've landed (so far) on:

```
[character]
success_symbol = "[▶︎](bold green) "
error_symbol = "[▶︎](bold red) "

[cmd_duration]
disabled = true

[directory]
format = "[$path]($style)[$read_only]($read_only_style) "
read_only = "✗"

[hostname]
format = "[$hostname]($style)[$ssh_symbol](bold red) in "
ssh_only = false
ssh_symbol = "◉"
style = "green"
trim_at = ""

[php]
symbol = "php"
version_format = "-${raw}"

[username]
format = "[$user]($style) on "
show_always = true
```

Some of symbol choices is due to unicode limitations on some systems. Still debugging that.

[1]: https://github.com/LucaFilipozzi/fish-theme-lfilipoz
[2]: https://github.com/robertgzr/porcelain
[3]: https://github.com/starship/starship
[4]: https://starship.rs/config/
