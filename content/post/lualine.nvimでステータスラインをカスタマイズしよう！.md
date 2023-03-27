+++
title = "lualine.nvimでステータスラインをカスタマイズしよう！"
date = 2023-03-17T22:33:33+09:00
tags = ["tech", "neovim"]
+++

この記事は[vim駅伝](https://vim-jp.org/ekiden/)3/17日の記事です。

neovimやvimを豪華に彩る中でおそらく必須となるであろうステータスライン系プラグインの紹介記事です。

現在公開されているプラグインは様々な種類がありますが、luaで書かれた中ではおそらくlualine.nvimがデファクトです。

https://github.com/nvim-lualine/lualine.nvim

***導入後***
![2023-03-17 13.17のイメージ.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2664731/52bf9aa3-3dc0-ffcd-2aae-7f8058578f35.jpeg)

筆者のコード

https://github.com/Liquid-system/NeovimConfig/blob/main/lua/plugins/lualine.lua

READMEが非常に丁寧に書かれているので、ステータスラインの位置を指定し、lualineに同梱されているコンポーネントを設定するのが定石ですが、筆者はlualine.nvimの`examples`ディレクトリにある`evil_lualine.lua`をカスタマイズしています。

https://github.com/nvim-lualine/lualine.nvim/blob/master/examples/evil_lualine.lua

改良すべき点
Lspを表示するコンポーネントです。`null-ls`などのフォーマット系プラグインを使用している場合は`client.name ~= "null-ls" then`とif文で除外するのがベターです。

```lua
ins_right {
      -- Lsp server name .
      function()
        local msg = "No Active"
        local buf_ft = vim.api.nvim_buf_get_option(0, "filetype")
        local clients = vim.lsp.get_active_clients()
        if next(clients) == nil then
          return msg
        end
        for _, client in ipairs(clients) do
          local filetypes = client.config.filetypes
          if filetypes and vim.fn.index(filetypes, buf_ft) ~= -1 and client.name ~= "null-ls" then
            return client.name
          end
        end
        return msg
      end,
      icon = " LSP:",
      color = { fg = colors.orange },
    }
```

#### 結論
lualine.nvimは比較的癖のないプラグインで扱いやすいです。皆さんもぜひステータスラインライフを送りましょう！
