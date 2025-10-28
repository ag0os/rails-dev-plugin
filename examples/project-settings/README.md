# Project Settings Examples

These are example `.claude/settings.json` files for different project types.

## Rails Projects

For Rails projects where you want the Rails Dev Plugin active:

```bash
# Copy to your Rails project root
mkdir -p .claude
cp examples/project-settings/rails-project.json .claude/settings.json
git add .claude/settings.json
git commit -m "Enable Rails Dev Plugin for this project"
```

**Content:** [rails-project.json](./rails-project.json)

This configuration:
- ✅ Automatically adds the rails-dev-plugin marketplace
- ✅ Installs the plugin for all team members
- ✅ Enables auto-install when teammates trust the repository

## Non-Rails Projects

For non-Rails projects where you want to disable the plugin:

```bash
# Copy to your non-Rails project root
mkdir -p .claude
cp examples/project-settings/non-rails-project.json .claude/settings.json
git add .claude/settings.json
git commit -m "Disable Rails Dev Plugin for this project"
```

**Content:** [non-rails-project.json](./non-rails-project.json)

This configuration:
- ✅ Disables the Rails Dev Plugin in this project
- ✅ Keeps the plugin installed globally but inactive here
- ✅ Prevents context pollution in non-Rails projects

## Quick Setup

### For Your Personal Templates Directory

If you want quick access to these templates:

```bash
# Create personal templates directory
mkdir -p ~/.config/claude-code/templates

# Copy examples
cp examples/project-settings/rails-project.json ~/.config/claude-code/templates/
cp examples/project-settings/non-rails-project.json ~/.config/claude-code/templates/

# Use in new projects
cp ~/.config/claude-code/templates/rails-project.json .claude/settings.json
```

### Shell Aliases (Optional)

Add to your `~/.zshrc` or `~/.bashrc`:

```bash
# Rails project setup
alias claude-rails='mkdir -p .claude && cp ~/.config/claude-code/templates/rails-project.json .claude/settings.json && echo "✅ Rails Dev Plugin enabled"'

# Non-Rails project setup
alias claude-disable-rails='mkdir -p .claude && cp ~/.config/claude-code/templates/non-rails-project.json .claude/settings.json && echo "✅ Rails Dev Plugin disabled"'
```

Then simply run:
```bash
# In any Rails project
claude-rails

# In any non-Rails project
claude-disable-rails
```

## Per-Project vs Team-Wide

### Individual Developer (You Only)
Use the simple disable configuration:
```json
{
  "plugins": {
    "disabled": ["rails-dev-plugin@ag0os"]
  }
}
```

### Team Distribution (Whole Team)
Use the full marketplace + autoInstall configuration:
```json
{
  "plugins": {
    "marketplaces": [
      {
        "name": "rails-dev",
        "source": "ag0os/rails-dev-plugin"
      }
    ],
    "installed": ["rails-dev-plugin@rails-dev"],
    "autoInstall": true
  }
}
```

## Documentation

For more details about using this plugin, see:
- [Quick Start Guide](../../docs/quick-start.md)
- [Agent Decision Tree](../../docs/agent-decision-tree.md)
- [Main README](../../README.md)
