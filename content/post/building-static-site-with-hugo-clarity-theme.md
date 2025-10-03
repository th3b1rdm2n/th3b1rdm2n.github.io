---
author: "Bírd Màn"
title: "Building Static Site with Hugo Clarity Theme"
date: "2025-09-26"
description: "A guide to building elegant static sites."
summary: "This guide outlines the process of building and deploying a static website using Hugo and the Clarity theme, managed via Git and automatically deployed to GitHub Pages with a custom domain."
featured: true
tags: ["Technology", "Engineering"]
categories: ["Research"]
aliases: ["building-static-site-with-hugo-clarity-theme"]
thumbnail: "images/banner-building-static-site-with-hugo-clarity-theme.webp"
---
Static site generators like Hugo allow you to build fast, secure, and highly customizable websites without the overhead of server-side logic. Whether you're creating a personal blog, research notebook, or engineering portfolio, this guide provides a complete, automation-focused setup for a professional web presence.

#### Environment Setup (Installation Scripts)
The development environment requires several tools: Go (the required runtime for building Hugo from source and for certain features), Hugo(the core static site generator, including Sass/SCSS and other advanced features), and essential development tools (git, gh, pre-commit utilities).

_Installing go_  
We begin with a Bash script to install Go programming language. The script performs architecture detection, checks for existing versions, downloads the Go archive, installs it to a local directory, and ensures your shell environment is updated to include Go in the system path.
```shell

# Golang Installation
install_go() {
  # Determine OS and architecture
  local os arch
  case "$(uname -s)" in
    Linux*) os="linux" ;;
    Darwin*) os="darwin" ;;
    *) echo "Unsupported OS"; return 1 ;;
  esac
  case "$(uname -m)" in
    x86_64*) arch="amd64" ;;
    arm64*) arch="arm64" ;;
    *) echo "Unsupported architecture"; return 1 ;;
  esac

  # Get the latest Go version
  local version
  version=$(curl -fsS https://go.dev/VERSION?m=text | head -n1) || {
    echo "Failed to get Go version"
    return 1
  }

  # Validate version string
  if [[ -z "$version" || ! "$version" =~ ^go[0-9]+\.[0-9]+(\.[0-9]+)?$ ]]; then
    echo "Invalid Go version string: '$version'"
    return 1
  fi

  # Check if Go is already installed with the same version
  if command -v go &> /dev/null && [[ "$(go version)" =~ $version ]]; then
    echo "Go $version is already installed"
    return 0
  fi

  # Set archive path and download URL
  local filename="${version}.${os}-${arch}.tar.gz"
  local archive_path="$HOME/Downloads/$filename"
  local url="https://go.dev/dl/$filename"

  # Download archive if not already cached
  if [[ -f "$archive_path" ]]; then
    echo "Using cached archive: $archive_path"
  else
    echo "Downloading from: $url"
    wget --secure-protocol TLSv1_3 -O "$archive_path" "$url" || {
      echo "Download failed"
      return 1
    }
  fi

  # Remove existing Go and extract new version
  rm -rf "$HOME/.local/go"
  tar -C "$HOME/.local" -xzf "$archive_path" || {
    echo "Extraction failed"
    return 1
  }

  # Update PATH in shell config
  local shell_config
  shell_config=$(find "$HOME" -maxdepth 1 \( -name ".bash_aliases" -o -name ".zshrc" \) -print -quit)
  if [[ -z "$shell_config" ]]; then
    echo "No shell configuration file found"
    return 1
  fi

  if ! grep -q 'export PATH=.*go/bin' "$shell_config"; then
    echo 'export PATH="$HOME/.local/go/bin:$PATH"' >> "$shell_config"
    echo "Updated PATH in $shell_config"
  fi

  export PATH="$HOME/.local/go/bin:$PATH"
  echo "Go $version installed in $HOME/.local/go"
  echo "Run 'source $shell_config' or restart your shell to use Go"
  go version || echo "Go not found in PATH"
}
install_go

```
_Installing hugo_  
Then we'll install Hugo (v0.150.0) using the official .deb files from the Hugo GitHub repository. The Bash function defines the Hugo version, architecture, and proceeds to downloading the standard and extended Hugo .deb files if they don't already exist locally and installs them using the Debian package manager.
```shell
# Hugo Installation
install_hugo() {
  local Version="0.150.0"
  local Arch="amd64"
  local OS="linux"
  local base_url="https://github.com/gohugoio/hugo/releases/download/v${Version}"

  local standard_deb="hugo_${Version}_${OS}-${Arch}.deb"
  local extended_deb="hugo_extended_${Version}_${OS}-${Arch}.deb"

  if [ ! -f "${standard_deb}" ] || [ ! -f "${extended_deb}" ]; then
    echo "Downloading Hugo v${Version}..."
    wget --secure-protocol TLSv1_3 "${base_url}/${standard_deb}"
    wget --secure-protocol TLSv1_3 "${base_url}/${extended_deb}"
  else
    echo "Hugo v${Version} DEBs already downloaded."
  fi

  echo "Installing Hugo..."
  sudo apt install -y -f ./"${standard_deb}" ./"${extended_deb}"
}
install_hugo
```
_Installing imagemagick, git, gh, and pre-commit_  
Finally, we'll use the Debian package manager (apt) to install essential development tools. ImageMagick(`magick`) provides powerful utilities for image conversion and favicon/logo generation, which we’ll use later when customizing the Hugo site. Git(`git`) manages version control and theme integration, while the GitHub CLI (`gh`) streamlines repository setup and deployment. We’ll also install Python utilities (python3-pip, python3-venv, pipx) for virtual environment management and installing tools like `pre-commit`, which enforces code quality and security checks before commits.
```shell
# enter sudo user password when prompted
install_tools_via_package_manager() {
  local pkg_manager
  local install_cmd
  local tools="imagemagick git gh python3-pip python3-venv pipx"

  if [ -f /etc/os-release ]; then
    source /etc/os-release
    case $ID in
      debian | ubuntu)
        pkg_manager="apt"
        install_cmd="sudo apt -y install"
        sudo apt update
        ;;
      centos | rhel | fedora)
        pkg_manager="dnf"
        install_cmd="sudo dnf -y install"
        ;;
      opensuse)
        pkg_manager="zypper"
        install_cmd="sudo zypper -y install"
        ;;
      *)
        echo "Unsupported Linux distribution ($ID)."
        return 1
        ;;
    esac
  else
    echo "Could not detect OS."
    return 1
  fi

  echo "Installing $tools using $pkg_manager..."
  if ! $install_cmd $tools; then
    echo "Failed to install tools."
    return 1
  fi
  
  # Ensure pipx environment is ready
  if ! command -v pipx &> /dev/null; then
    echo "pipx not found. Aborting pre-commit installation."
    return 1
  fi

  echo "Installing pre-commit using pipx..."
  if ! pipx install pre-commit; then
    echo "Failed to install pre-commit."
    return 1
  fi
}
install_tools_via_package_manager

```
#### Project Initialization & Configuration
This section, uses the installed gh, git and hugo tools to set up the repository, site structure, and initial configuration.

_Define Core Variables_  
Start by defining the necessary environment and project variables. This centralizes all configurations for the repository, Hugo project, and custom domain. These variables will be used throughout the process.
```shell
# Git variables
GIT_USER_NAME="th3b1rdm2n"
GIT_USER_HOMEPAGE="https://github.com/$GIT_USER_NAME"
GIT_REPO_NAME="$GIT_USER_NAME.github.io"
GIT_REPO_URL="https://github.com/$GIT_USER_NAME/$GIT_REPO_NAME.git"
GIT_REPO_DESCRIPTION="A Digital Nest for Learning, Living and Leading with Intent."
GIT_MAIN_BRANCH="production"
GIT_DEV_BRANCH="development"
GIT_REMOTE_NAME="origin"

# Hugo variables
HUGO_SITENAME="$GIT_REPO_NAME"
HUGO_PROJECT_ROOT="$HOME/$HUGO_SITENAME"
HUGO_THEME_REPO_URL="https://github.com/chipzoller/hugo-clarity"
HUGO_THEME_DIR="themes/$(basename "$HUGO_THEME_REPO_URL")"

# Custom variables
CUSTOM_DOMAIN="th3b1rdm2n.site"

```
_Initialize the Project_  
Next, we'll authenticate to GitHub using the `gh` command-line tool and create a new public repository.
```shell
# Authenticate to Github and create remote repository
gh auth status # Check authentication status
gh auth login -h github.com -p https -w # Authenticate to GitHub
gh auth setup-git # Configure Git to use GitHub CLI as credential helper
gh repo create "$GIT_REPO_NAME" --public --homepage "$GIT_USER_HOMEPAGE" --description "$GIT_REPO_DESCRIPTION" # Create remote repo
```
We'll then initialize the local Git repository, create a new Hugo site, add the Clarity theme as a submodule, and push to remote repository.
```shell

# Set project root and clone the theme repository
cd $(dirname "$HUGO_PROJECT_ROOT")
hugo new site "$HUGO_SITENAME" --format yaml
cd "$HUGO_SITENAME"
hugo mod init "$HUGO_SITENAME"

# Initialize Git 
git init . # Initialize local Git repo
git config --local user.name "$GIT_USER_NAME" # Set local Git username
git config --local user.email "" # Set local Git email
git branch -M "$GIT_MAIN_BRANCH" # Rename default branch to production
git submodule add $HUGO_THEME_REPO_URL "$HUGO_THEME_DIR"  # add a hugo theme as submodule
git commit --allow-empty -m "chore: initial commit - $(date +%F_%H:%M)" # initial empty commit
git remote add "$GIT_REMOTE_NAME" "$GIT_REPO_URL" # Add remote origin
git push -u "$GIT_REMOTE_NAME" "$GIT_MAIN_BRANCH" # Push production branch to remote
```
_Customize the Project_  
We'll then configure pre-commit hooks to ensure code quality, add a .gitignore file to exclude unnecessary files, add a CNAME file for `gh-pages` to use custom domain, copy the theme's exampleSite and icons folder into the project root and static folders respectively, update the images of the icons folder, and modification to various files as shown in the walkthrough video.
{{< youtube 9x-HFRXL2yM >}}

```shell

# Create pre-commit config file
cat > "$HUGO_PROJECT_ROOT/.pre-commit-config.yaml" <<EOF
repos:
- repo: https://github.com/pre-commit/pre-commit-hooks
  rev: v5.0.0
  hooks:
  - id: check-added-large-files
    args: ["--maxkb=102400", "--enforce-all"]
- repo: https://github.com/zricethezav/gitleaks
  rev: v8.18.0
  hooks:
  - id: gitleaks
EOF
pre-commit install # Install pre-commit hooks
pre-commit autoupdate # Update hook repos to latest version
pre-commit run --all-files # Run pre-commit hooks on all files

# add .gitignore file
cat > "$HUGO_PROJECT_ROOT/.gitignore" <<EOF
# IDE
**/.code-workspace
**/.idea
**/.vscode/
!**/.vscode/extensions.json
!**/.vscode/settings.json

# Go
**/*.out
**/*.test
**/vendor/

# Hugo
**/resources
**/public

# JavaScript
**/.angular/
**/.next/
**/.node_repl_history
**/.npm
**/.nuxt/
**/.parcel-cache/
**/.temp
**/.vue-cli-service/
**/*.js.map
**/*.min.js
**/*.ts.map
**/*.tsbuildinfo
**/coverage/
**/dist/
**/e2e/
**/node_modules/
**/npm-debug.log
**/package-lock.json
**/public/
**/yarn-debug.log
**/yarn-error.log

# Miscellaneous
**/*.bak
**/*.log
**/*.swp
**/.env
EOF

# Add CNAME file at the project root for custom domain mapping
echo -e "$CUSTOM_DOMAIN\nwww.$CUSTOM_DOMAIN" > "$HUGO_PROJECT_ROOT/CNAME"

# Use the exampleSite in developing and modifying yours
cp -a "$HUGO_THEME_DIR"/exampleSite/* $HUGO_PROJECT_ROOT/ # Copy theme's exampleSite content into the root of your Hugo project
cp -a "$HUGO_THEME_DIR"/static/icons $HUGO_PROJECT_ROOT/static/ # Copy theme's static icons to your project's static directory

# Create a logo - to see list run: `convert -list font | less`
generate_ssg_image() {
  local input_image="" output_dir="" logo_text=""

  # Parse CLI arguments
  while [[ "$#" -gt 0 ]]; do
    case "$1" in
      -i|--input) input_image="$2"; shift 2 ;;
      -o|--output) output_dir="$2"; shift 2 ;;
      -l|--logo) logo_text="$2"; shift 2 ;;
      -*|*) echo "Usage: $FUNCNAME -i <input_image> -o <output_dir>"; return 1 ;;
    esac
  done

  # Validate input
  [[ -z "$input_image" || -z "$output_dir" ]] && {
    echo "Usage: $FUNCNAME -i <input_image> -o <output_dir>"; return 1; }
  [[ ! -f "$input_image" ]] && { echo "File not found: $input_image"; return 1; }

  mkdir -p "$output_dir"

  local transparent_image="$output_dir/th3b1rdm2n.png"
  convert "$input_image" -fuzz 10% -transparent black "$transparent_image"

  declare -A output_names=(
    [16]="favicon-16x16.png"
    [32]="favicon-32x32.png"
    [150]="mstile-150x150.png"
    [180]="apple-touch-icon.png"
    [192]="android-chrome-192x192.png"
    [256]="android-chrome-256x256.png"
    [512]="android-chrome-512x512.png"
  )

  echo "Generating icons from $transparent_image..."
  for size in "${!output_names[@]}"; do
    convert "$transparent_image" -resize "${size}x${size}" "$output_dir/${output_names[$size]}"
    echo "Generated: ${output_names[$size]}"
  done

  convert "$output_dir/favicon-16x16.png" "$output_dir/favicon.ico"
  
  # Create a logo - to see list run: `convert -list font | less`
  convert -size 140x38 xc:none -gravity center -stroke white -strokewidth 2 -fill black -pointsize 24 -font Liberation-Sans -annotate +0+0 "$logo_text" "$(dirname "$output_dir")/logos/logo.png"
  echo "Done: All icons saved to $output_dir"
}
generate_ssg_image -i "$HOME/Downloads/th3b1rdm2n.png" -o "$HUGO_PROJECT_ROOT/static/icons" -l "bírd màn"
```
#### Git Workflow and Deployment
_Branching and Initial Deployment_  
A structured Git workflow ensures that development work is isolated from the live production code. This step creates a development branch for ongoing changes and then merges to the production branch, triggering the configured GitHub Actions workflow to build and deploy your site. After the initial push, navigate to your repository settings to enable GitHub Actions as the deployment source for GitHub Pages.
```shell
git checkout -b "$GIT_DEV_BRANCH" # Create and switch to main branch
git add .  # add the modified and populated files
git commit -m "chore: modified and populated site files - $(date +%F_%H:%M)"
git push -u "$GIT_REMOTE_NAME" "$GIT_DEV_BRANCH"

# Navigate to $GIT_USER_HOMEPAGE/$GIT_REPO_NAME/settings/pages. Switch the Source to `GitHub Actions`. Then run:
git switch "$GIT_MAIN_BRANCH"
git merge "$GIT_DEV_BRANCH"
git push -u "$GIT_REMOTE_NAME" "$GIT_MAIN_BRANCH" # Push main branch to remote
git switch "$GIT_DEV_BRANCH" # Switch back to development branch

```
_Configuring DNS Records_  
To make your site accessible via your custom domain, you must update your DNS settings. At your domain registrar (e.g., [Namecheap](https://namecheap.com)), buy a domain of your choice, navigate to Advanced DNS and add the following records to point your domain to the correct GitHub Pages IP addresses and repository URL.
```shell
open "https://ap.www.namecheap.com/Domains/DomainControlPanel/$CUSTOM_DOMAIN/advancedns"
```
Click on `Add New Record` and add A and CNAME records:
| Type      | Host | Value              | TTL       |
|-----------|------|--------------------|-----------|
| A Record  | @    | 185.199.108.153    | Automatic |
| A Record  | @    | 185.199.109.153    | Automatic |
| A Record  | @    | 185.199.110.153    | Automatic |
| A Record  | @    | 185.199.111.153    | Automatic |
| CNAME Record  | @    | <your $GIT_REPO_NAME value>   | Automatic |
```shell
open $GIT_USER_HOMEPAGE/$GIT_REPO_NAME/settings/pages 
```
With DNS records set, finalize the connection in GitHub. Enter <your $CUSTOM_DOMAIN value> in the Settings > Pages > Custom domain input field. GitHub will provision the necessary SSL certificate. Your Hugo site is now live and accessible via your custom domain.
