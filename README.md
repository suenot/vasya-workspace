# vasya-workspace

Workspace for the VasyaApp ecosystem. Each project is a git submodule.

| Repo | Visibility | Purpose |
|------|-----------|---------|
| [vasyaapp](https://github.com/suenot/vasyaapp) | public | The Telegram Client for Deep Focus |
| [vasyaapp-native](https://github.com/suenot/vasyaapp-native) | public | Native GPUI / Iced desktop clients on a shared Rust engine |
| [vasya-landing](https://github.com/suenot/vasya-landing) | public | Landing page |
| [vasya-landing-astro](https://github.com/suenot/vasya-landing-astro) | public | Landing page (Astro) |
| sdvg-vasyapp | private | Private, needs access |
| vasya-backend | private | Private, needs access |

## Setup

```bash
git clone --recurse-submodules git@github.com:suenot/vasya-workspace.git
```

Private submodules fail to clone without access; that is expected.
