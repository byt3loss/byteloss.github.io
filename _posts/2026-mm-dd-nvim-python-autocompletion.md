Install the Pyright LSP

```sh
sudo apt install nodejs npm
sudo npm install -g pyright
```

Edit the file `~/.config/nvim/lua/plugins/lsp.lua` and add the following lua code.

```lua
return {
  {
    "neovim/nvim-lspconfig",
    config = function()
      vim.lsp.config("pyright", {
        cmd = { "pyright-langserver", "--stdio" },

        filetypes = { "python" },

        root_markers = {
          "pyproject.toml",
          "pyrightconfig.json",
          "setup.py",
          "setup.cfg",
          "requirements.txt",
          "Pipfile",
          "pyenv.cfg",
          ".venv",
          ".git",
        },

        settings = {
          python = {
            analysis = {
              autoSearchPaths = true,
              useLibraryCodeForTypes = true,
              diagnosticMode = "openFilesOnly",
            },
          },
        },
      })

      vim.lsp.enable("pyright")
    end,
  },
}
```
