# Requirements

This will be a Rust project that creates a binary that, once launched inside an Arch ISO shell, will end up the environment fully customized as I wish.

## Delivery

- A file named `package.conf` that contains a list of package names to install
- A directory that contains all my dotfiles (stow-compliant)
- A binary file that is the built of this Rust repo

## Deploy

This project will be deployed in GitHub Releases, so that the following commands should completely configure correctly a brand new computer.

```bash
# download the binary file
curl -LfsSL -o nirioly "https://github.com/yourusername/nirioly/releases/latest/download/nirioly"
# make it executable
chmod +x nirioly
# launch it
./nirioly
```

## Secrets handling

It should handle secrets as input either via a file (e.g. `.env`) or by passing secret info via env vars (e.g. `NIRIOLY_PSW=super-secret`)

## Including files and dir directly inside the binary

- for file use the builtin macro `include_str!`
- for directory there is a popular crate called `include_dir`
