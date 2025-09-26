# gh-submodule-bury

A GitHub CLI extension that "buries" git submodules by converting them into regular directories while preserving their content.

## What it does

This tool removes a git submodule and adds its contents directly to the parent repository as regular files. It's useful when you want to:

- Stop using a submodule and incorporate its code directly into your repository
- Flatten a repository structure by removing submodule dependencies  
- Convert submodules to regular directories for easier maintenance

## Installation

```sh
gh extension install srz-zumix/gh-submodule-bury
```

## Usage

```sh
gh submodule-bury <submodule-path> [options]
```

### Options

- `-f, --force`: Force cleanup by automatically resetting uncommitted changes and removing untracked/ignored files
- `-h, --help`: Show help message

### Examples

```sh
# Bury a submodule at path "vendor/library"
gh submodule-bury vendor/library

# Force bury a submodule (automatically clean uncommitted changes)
gh submodule-bury --force vendor/library
```

### Important Notes

**Before burying a submodule**, ensure the following requirements are met:

- **Submodule must be initialized**: Run `git submodule update --init <path>` if needed
- **Clean working directory**: No uncommitted changes (staged or unstaged)  
- **No untracked files**: All files must be either committed or ignored
- **No .gitignore'd files**: Ignored files in submodule could cause issues when moved to parent repository

**If the submodule is not initialized**, run:

```sh
# Initialize a specific submodule
git submodule update --init path/to/submodule

# Or initialize all submodules
git submodule update --init --recursive
```

**If the submodule has uncommitted changes**, you'll need to either:

```sh
# Option 1: Commit your changes
cd path/to/submodule
git add .
git commit -m "Your commit message"

# Option 2: Discard changes (⚠️ This will lose your work!)
cd path/to/submodule
git reset --hard
git clean -fd
```

**If the submodule has .gitignore'd files**, you'll need to either:

```sh
# Option 1: Remove ignored files (⚠️ This will permanently delete them!)
cd path/to/submodule
git clean -fdX

# Option 2: Force add them if they should be tracked
cd path/to/submodule
git add -f <ignored-files>
git commit -m "Add previously ignored files"

# Option 3: Add them to parent repository's .gitignore first
echo "path/to/former-submodule/ignored-file" >> .gitignore
git add .gitignore
git commit -m "Add gitignore for former submodule files"
```

### Force Mode (`--force` option)

The `--force` option automatically resolves common issues by:

- **Resetting uncommitted changes**: Runs `git reset --hard HEAD`
- **Removing untracked files**: Runs `git clean -fd`
- **Removing ignored files**: Runs `git clean -fdX`

⚠️ **Warning**: Using `--force` will **permanently delete** all uncommitted work, untracked files, and ignored files in the submodule and its nested submodules.

```sh
# This will automatically clean everything and proceed
gh submodule-bury --force path/to/messy/submodule
```

Use `--force` only when you're certain you don't need any of the uncommitted changes or untracked files.

**Nested Submodules**: If your submodule contains nested submodules:

- **All nested submodules must be initialized**: Use `git submodule update --init --recursive` to ensure all levels are initialized
- **Recursive checking**: They will be checked recursively for uncommitted changes
- **Same requirements apply**: Each nested submodule must have a clean working directory

If you encounter issues with nested submodules, initialize them recursively:

```sh
# Initialize all nested submodules recursively  
git submodule update --init --recursive path/to/submodule
```

## What happens when you bury a submodule

1. **Backup**: Creates a temporary backup of the submodule contents
2. **Git hash capture**: Captures SHA1 hashes of all git-tracked files for content verification
3. **Deinitialize**: Runs `git submodule deinit` on the specified path
4. **Cache cleanup**: Removes nested submodules and main submodule from git cache using `git rm --cached`
5. **Remove**: Removes the submodule entry from git index and `.gitmodules`
6. **First commit**: Commits the submodule removal
7. **Restore**: Copies the backed-up contents back as regular files (preserving symbolic links)
8. **Cleanup**: Removes nested `.gitmodules` files and `.git` directories to clean up submodule artifacts
9. **Stage**: Adds the new files to the git index
10. **Second commit**: Commits the submodule contents as regular files
11. **Final verification**: Verifies that committed files have identical git hashes to original tracked files

The process automatically creates two separate commits:

- **First commit**: Records the removal of the submodule
- **Second commit**: Records the addition of the former submodule's contents as regular files

This separation makes it easy to track the transformation and provides a clear history of what happened.

## Safety features

- Validates that the specified path is actually a git submodule
- **Initialization validation**: Ensures the submodule and all nested submodules are properly initialized
- **Checks for uncommitted changes**: Ensures the submodule has no uncommitted changes or untracked files (or automatically cleans with `--force`)
- **Nested submodule support**: Recursively checks nested submodules for proper initialization and clean status
- **Force cleanup option**: `--force` flag provides automatic resolution of common issues
- **Git cache cleanup**: Uses `git rm --cached` to properly remove submodules (including nested ones) from git index
- **Nested .gitmodules cleanup**: Automatically removes `.gitmodules` files and `.git` directories from nested submodules
- **Git content integrity verification**: Compares git hash-objects of original and final committed files to ensure identical content
- Creates a backup of submodule contents before making any changes
- Provides detailed output showing what's happening at each step
- Preserves original submodule information for reference
- **Symbolic link preservation**: Preserves symbolic links as links rather than copying target files
- Separate commits for submodule removal and content addition

## Requirements

- Git repository with initialized submodules
- GitHub CLI (`gh` command)
- Unix-like environment (Linux, macOS, WSL)
