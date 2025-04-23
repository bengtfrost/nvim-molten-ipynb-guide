````markdown
# Guide: Using Molten-Nvim Directly with .ipynb Files (JSON View)

This guide details a workflow for interacting with Jupyter kernels and `.ipynb` notebook files directly within Neovim, using the [Molten-Nvim](https://github.com/benlubas/molten-nvim) plugin. The focus is on working with the raw JSON view that Neovim presents when opening a `.ipynb` file directly.

This method was tested on Linux Fedora 42 (Sway compositor, Foot terminal) with Neovim 0.11-dev, targeting a Python kernel running in a dedicated virtual environment.

**Key Finding:** While direct cell-based commands (`:MoltenEvaluateCell`, `:MoltenReevaluateCell`) may not work reliably when the cursor isn't within the source code lines of the raw JSON structure, **line-based and selection-based execution works well** after initialization and kernel connection. The `:MoltenImportOutput` command can be used to view results stored within the notebook file.

## Goal

To open a `.ipynb` file in Neovim, display its stored outputs, connect to a specific Python kernel (e.g., running from a `venv`), and execute code line-by-line or by selection directly from the JSON view.

## Dependencies & Setup

Correctly setting up dependencies across the **Neovim host environment** and the **kernel's virtual environment** is crucial.

### 1. System Dependencies (Fedora Example - for Neovim Host)

These packages provide tools needed by Neovim itself and the Python code run by plugins like Molten. Install using your system package manager (e.g., `dnf` on Fedora):

```bash
sudo dnf install neovim python3-neovim python3-jupyter-client nbformat
```
````

- **`neovim`**: The editor.
- **`python3-neovim`**: Provides the `pynvim` library for the system Python. Essential for Neovim <-> Python plugin communication (used by Molten).
- **`python3-jupyter-client`**: Provides core Jupyter communication libraries needed by the host. Required for stable operation and potentially `:UpdateRemotePlugins`.
- **`nbformat`**: Library required by Molten to parse `.ipynb` files (used by `:MoltenImportOutput`).

**Verification:** Run `:checkhealth provider` inside Neovim and check the `python3` section to confirm which Python executable Neovim is using for its host (e.g., `/usr/bin/python3`) and ensure these packages are installed and accessible for _that_ Python instance.

### 2. Kernel Virtual Environment Dependencies & Setup

This setup is for the specific Python environment where your notebook code will actually run (e.g., a project `venv`).

```bash
# 1. Activate your virtual environment (replace with your path)
# Example:
source ~/path/to/your/venv/bin/activate

# 2. Install required packages (using pip, uv, or your preferred tool)
# Example:
pip install ipykernel notebook camel-ai # Add any other libraries your code needs

# 3. Register the KernelSpec (makes it discoverable by Jupyter/Molten)
# Run this command while the virtual environment is active:
# python -m ipykernel install --user --name=<kernel-name> --display-name "Python (<display-name>)"
# Example:
python -m ipykernel install --user --name=camel-ai --display-name "Python (camel-ai)"
```

- **`ipykernel`**: Allows this environment to act as a Jupyter kernel.
- **`notebook`**: Provides tools like `jupyter console`, `jupyter lab`, `jupyter kernel` etc., for starting kernels.
- **`camel-ai`** (Example): Your project's specific dependencies.
- **Registering the KernelSpec** is highly recommended for easy kernel selection by name in various frontends (including Molten).

### 3. Neovim Plugin Installation & Configuration

- Install `molten-nvim` and its dependency `nvim-treesitter` using your plugin manager (e.g., `lazy.nvim`).
- **Crucially:** Run `:UpdateRemotePlugins` in Neovim _once_ after installing `molten-nvim` and **restart Neovim**. This registers Molten's Python components.
- Use the following example `molten-nvim` configuration (save as `lua/plugins/molten.lua` or adapt for your setup):

  ```lua
  -- lua/plugins/molten.lua
  -- Example configuration for Molten-Nvim
  return {
    "benlubas/molten-nvim",
    version = "^1", -- Use version pinning recommended by the author
    -- Lazy load based on filetype; keymaps are deferred via VimEnter anyway
    ft = { "python", "markdown", "quarto", "json" }, -- Added 'json' for .ipynb files
    dependencies = {
      { "nvim-treesitter/nvim-treesitter", build = ":TSUpdate", lazy=true },
      { "nvim-tree/nvim-web-devicons", lazy=true }, -- Optional: for icons
    },
    build = ":UpdateRemotePlugins", -- Run UpdateRemotePlugins post-install/update
    init = function()
      -- Options set before plugin loads (optional)
      vim.g.molten_auto_open_output = true -- Show output automatically
      -- vim.g.molten_virt_text_output = true -- Use virtual text for output (alternative)
      -- vim.g.molten_output_win_max_height = 15 -- Control output window height
    end,
    config = function()

      -- Helper function to run Molten commands safely
      local function run_molten_command(cmd_name, args_str, success_msg, error_msg)
        args_str = args_str or ""
        local full_cmd = cmd_name .. " " .. args_str
        local cmd_exists_name = ":" .. cmd_name

        vim.defer_fn(function() -- Delay execution slightly
            if vim.fn.exists(cmd_exists_name) == 2 then -- Check if command exists *before* running
              vim.cmd(full_cmd)
              if success_msg then vim.notify(success_msg, vim.log.levels.INFO, { title = "Molten" }) end
            else
              local default_error = "Command '" .. cmd_exists_name .. "' not available. Molten init incomplete or failed?"
              vim.notify(error_msg or default_error, vim.log.levels.ERROR, { title = "Molten" })
            end
        end, 50) -- 50ms delay
      end

      -- Helper function for visual mode evaluation
      local function run_molten_visual_evaluate()
          local cmd_exists_name = ":MoltenEvaluateVisual"
          vim.defer_fn(function()
              if vim.fn.exists(cmd_exists_name) == 2 then
                  -- Use nvim_command for reliable execution of range command in callback
                  vim.api.nvim_command("normal! '<,'>:MoltenEvaluateVisual<CR>")
                  vim.notify("Evaluating selection...", vim.log.levels.INFO, { title = "Molten" })
                  -- Exit visual mode
                  vim.api.nvim_feedkeys(vim.api.nvim_replace_termcodes("<Esc>", true, false, true), 'n', true)
              else
                  vim.notify("Command ':MoltenEvaluateVisual' not available.", vim.log.levels.ERROR, { title = "Molten" })
                  vim.api.nvim_feedkeys(vim.api.nvim_replace_termcodes("<Esc>", true, false, true), 'n', true)
              end
          end, 50)
      end

      -- Define Keymaps after Neovim is fully loaded (using VimEnter)
      local group = vim.api.nvim_create_augroup("MoltenPostLoadKeymaps", { clear = true })
      vim.api.nvim_create_autocmd("VimEnter", {
        group = group, pattern = "*", desc = "Define Molten keymaps after plugins likely initialized", once = true,
        callback = function()
          local map = vim.keymap.set
          local leader = "<leader>" -- Assumes leader key is set globally

          -- Initialization / Kernel Selection
          map("n", leader .. "ji", function() run_molten_command("MoltenInit", nil, "Molten Initializing...") end, { desc = "Molten: Initialize/Attach Kernel", noremap = true, silent = true })
          map("n", leader .. "jca", function() run_molten_command("MoltenSelectKernel", nil, "Select Kernel...") end, { desc = "Molten: Select Kernel", noremap = true, silent = true })

          -- Evaluate (Line/Selection focused for .ipynb JSON view)
          map("n", leader .. "jl", function() run_molten_command("MoltenEvaluateLine", nil, "Evaluating line...") end, { desc = "Molten: Evaluate Line", noremap = true, silent = true })
          map("v", leader .. "js", run_molten_visual_evaluate, { desc = "Molten: Evaluate Selection", noremap = true, silent = true })
          map("n", leader .. "jr", function() run_molten_command("MoltenReevaluateCell", nil, "Re-evaluating cell (?)") end, { desc = "Molten: Re-evaluate Cell (?)", noremap = true, silent = true})

          -- Cell Commands (NOTE: Use with caution in raw .ipynb JSON view)
          map("n", leader .. "jc", function() run_molten_command("MoltenEvaluateCell", nil, "Evaluating cell...") end, { desc = "Molten: Evaluate Cell (?)", noremap = true, silent = true })
          map("n", leader .. "jj", function() run_molten_command("MoltenEvaluateCell", nil, "Evaluating cell..."); run_molten_command("MoltenNextCell") end, { desc = "Molten: Evaluate Cell & Next (?)", noremap = true, silent = true })

          -- Navigation (NOTE: Use with caution in raw .ipynb JSON view)
          map("n", leader .. "jn", function() run_molten_command("MoltenNextCell") end, { desc = "Molten: Next Cell (?)", noremap = true, silent = true })
          map("n", leader .. "jp", function() run_molten_command("MoltenPrevCell") end, { desc = "Molten: Previous Cell (?)", noremap = true, silent = true })

          -- Output/Kernel Management
          map("n", leader .. "jo", function() run_molten_command("MoltenClearOutput") end, { desc = "Molten: Clear Cell Output", noremap = true, silent = true })
          map("n", leader .. "jk", function() run_molten_command("MoltenInterruptKernel", nil, "Interrupting kernel...") end, { desc = "Molten: Interrupt Kernel", noremap = true, silent = true })
          map("n", leader .. "jR", function() run_molten_command("MoltenRestartKernel", nil, "Restarting kernel...") end, { desc = "Molten: Restart Kernel", noremap = true, silent = true })

          -- Register WhichKey group
          local wk_status_ok, wk = pcall(require, "which-key")
          if wk_status_ok then
              wk.register({ { "", group = "Molten" } }) -- Register group name
          end
        end, -- End VimEnter callback
      }) -- End autocmd definition

      -- Optional: Autocommand for syntax adjustments in .ipynb JSON view
      vim.api.nvim_create_autocmd("FileType", {
        pattern = "json", -- Trigger on json for .ipynb files
        group = vim.api.nvim_create_augroup("MoltenCustomJsonSyntax", { clear = true }),
        callback = function()
          -- Add any custom syntax matching or concealing for JSON here if desired
          -- Example: Conceal quotes around keys (use cautiously)
          -- vim.cmd([[syntax match JsonKey #"\v(\w+)"\ze\s*:# contains=@NoSpell conceal cchar= ]])
        end,
      })

    end, -- End config function
  } -- End return table
  ```

## Execution Workflow

1.  **Start the Kernel:** Activate the correct venv (e.g., `camel-ai`) and start the kernel process using **one** of these methods:

    - `jupyter kernel --kernel <kernel-name>` (e.g., `jupyter kernel --kernel camel-ai`) - Starts only the kernel process in the background.
    - `jupyter console --kernel <kernel-name>` - Starts kernel and an attached IPython console frontend.
    - `jupyter lab` or `jupyter notebook` - Start the web UI, open/create a notebook using the desired kernel.

2.  **Open Neovim & Notebook:**

    - `nvim path/to/your/notebook.ipynb` (Displays the raw JSON structure).

3.  **Display Stored Outputs & Connect Kernel:**

    - Run `:MoltenImportOutput` in the `.ipynb` buffer.
    - If prompted ("Need to Initialize..."), select your running kernel (e.g., `camel-ai` should be listed if the kernelspec was registered and the process is running). Wait for the "Kernel... is ready" confirmation.
    - _(Alternatively, manually run `<leader>ji` then `<leader>jca` before `:MoltenImportOutput`)_.

4.  **Execute Code from JSON View:**

    - **Single Line:** Place cursor on a line within the `"source": [...]` array -> Run `<leader>jl` (or `:MoltenEvaluateLine`).
    - **Selection:** Visually select lines within `"source": [...]` array -> Run `<leader>js` (or `:MoltenEvaluateVisual`).
    - **Re-evaluate:** Place cursor on a line within `"source": [...]` -> Run `<leader>jr` (or `:MoltenReevaluateCell`). _(Note: The context for what "cell" means to this command in JSON view might be the line itself or the containing block)._

5.  **Important Note on Cell Commands:** Standard cell-based commands like `<leader>jc` (`:MoltenEvaluateCell`), `<leader>jj`, `<leader>jn`, `<leader>jp` **may not work reliably or as expected** when operating directly on the raw `.ipynb` JSON view. Molten might not recognize the JSON structure as distinct cell boundaries for these commands. Focus on line, selection, and potentially re-evaluate commands in this workflow.

## Key Takeaways

- Molten-Nvim _can_ connect to kernels and execute code when viewing `.ipynb` files directly as JSON.
- Dependencies need to be correctly installed in _both_ the **Neovim host Python environment** (`python3-neovim`, `python3-jupyter-client`, `nbformat`) and the **kernel's virtual environment** (`ipykernel`, `notebook`, project libs).
- Execution from the JSON view works reliably with **line-based** (`:MoltenEvaluateLine`, `<leader>jl`) and **selection-based** (`:MoltenEvaluateVisual`, `<leader>js`) commands. `:MoltenReevaluateCell` (`<leader>jr`) may also work depending on cursor position.
- Cell-based commands (`:MoltenEvaluateCell`, etc.) might require a different file view or specific cursor positioning within the source lines.

## Related Projects & Technologies

[![Neovim Logo](https://img.shields.io/badge/Neovim-neovim.io-57A143?style=flat-square&logo=neovim&logoColor=white)](https://neovim.io/)
[![Python Logo](https://img.shields.io/badge/Python-python.org-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org/)
[![Jupyter Logo](https://img.shields.io/badge/Jupyter-jupyter.org-F37626?style=flat-square&logo=jupyter&logoColor=white)](https://jupyter.org/)
[![Molten-Nvim GitHub](https://img.shields.io/badge/Molten_Nvim-GitHub-57A143?style=flat-square&logo=neovim&logoColor=white)](https://github.com/benlubas/molten-nvim)

```

```
