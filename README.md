![](https://i.ibb.co/W4XFKYsd/Gemini-Generated-Image-dc2yo0dc2yo0dc2y.png)

# Claude Desktop + MCPs on Linux (The Atomic Way)

This guide presents a robust architecture for running **Claude Desktop** with full **Model Context Protocol (MCP)** support on Linux distributions, specifically focused on immutable/atomic systems (Fedora Silverblue, Kinoite, SteamOS) or any distro where keeping the base system clean is desired.

We use **Distrobox** with an **Arch Linux** container to ensure access to the latest tool versions, completely isolating the development environment from your main installation.

## The Goal

To create an **Augmented Personal Assistant** capable of:

1. Complex **reasoning** (Sequential Thinking).
2. **Navigating and extracting** data from the Web (Apify).
3. **Reading** your local Second Brain (Obsidian).
4. **Remembering** you long-term (Mem0).
5. Natively **integrating** with Google Drive, Gmail, and GitHub.

## Prerequisites

* A Linux distribution.
* `distrobox` installed.
* `curl` installed.
* An Anthropic account (Claude).
* API Keys for services (Apify, Mem0).

## Installation Step-by-Step

### 1. Create the Isolated Container

We will create an Arch Linux container with a **separate Home directory**. This prevents conflicts with your personal configurations and keeps the assistant's files organized.

```bash
# In your Host terminal
mkdir -p ~/distrobox_homes/mcp-servers

distrobox create -n mcp-servers -i archlinux:latest --home ~/distrobox_homes/mcp-servers

# Enter the container
distrobox enter mcp-servers

```

### 2. Install Dependencies & Base (Inside Container)

Inside the container, we install Node.js, Python, Git, and the necessary graphic libraries to run Electron applications (like Claude).

```bash
# 1. Update and install base tools
sudo pacman -Syu --noconfirm
sudo pacman -S --needed base-devel git nodejs npm python python-pip fuse2 --noconfirm

# 2. Install graphic libraries (Essential to avoid sandbox/GPU errors)
sudo pacman -S --noconfirm gtk3 nss alsa-lib libdrm mesa libxss at-spi2-core libnotify xdg-utils libappindicator-gtk3 libsecret

# 3. Install 'yay' (AUR Helper)
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si --noconfirm
cd ..
rm -rf yay-bin

```

### 3. Install Claude & Python MCPs

Still inside the container:

```bash
# Install Claude Desktop via AUR
yay -S claude-desktop-appimage --noconfirm

# Install Memory MCP Server (Mem0 requires Python)
# Note: Sequential and Apify run via npx and do not need prior installation
sudo pip install mem0-mcp-server --break-system-packages

```

### 4. Export to Host System

Export the shortcut so Claude appears in your main system's application menu.

```bash
distrobox-export --app claude-desktop
exit

```

## Configuration (On Host)

### 1. Adjust Permissions (Sandbox Fix)

Electron applications inside containers need the `--no-sandbox` flag to run. We must edit the exported shortcut.

1. Edit the `.desktop` file (Usually in `~/.local/share/applications/`):
```bash
nano ~/.local/share/applications/mcp-servers-claude-desktop.desktop

```


2. Locate the line starting with `Exec=` and add `--no-sandbox` before `%U`. Example:
```ini
Exec=/usr/bin/distrobox-enter -n mcp-servers -- /usr/bin/claude-desktop --no-sandbox %U

```



### 2. Restore the Icon

Since the container has a custom home, the host system might lose the icon reference.

```bash
mkdir -p ~/.local/share/icons/hicolor/512x512/apps/
curl -L -o ~/.local/share/icons/hicolor/512x512/apps/claude-desktop.png "https://uxwing.com/wp-content/themes/uxwing/download/brands-and-social-media/claude-ai-icon.png"
gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor/

```

### 3. The JSON File (The Brain)

Create the configuration file in the **custom** container home directory.

**Path:** `~/distrobox_homes/mcp-servers/.config/Claude/claude_desktop_config.json`

Copy the content below, replacing the `[YOUR_...]` placeholders with your actual data.

```json
{
  "globalShortcut": "",
  "mcpServers": {
    "Sequential Thinking": {
      "command": "/usr/bin/npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-sequential-thinking"
      ]
    },
    "Apify": {
      "command": "/usr/bin/npx",
      "args": [
        "-y",
        "@apify/actors-mcp-server",
        "--tools",
        "actors,docs,runs,storage,apify/rag-web-browser,fatihtahta/reddit-scraper-search-fast,starvibe/youtube-video-transcript,apify/instagram-profile-scraper,apify/instagram-post-scraper,apify/export-instagram-comments-posts,curious_coder/facebook-ads-library-scraper,curious_coder/linkedin-jobs-scraper,supreme_coder/linkedin-post"
      ],
      "env": {
        "APIFY_TOKEN": "[YOUR_APIFY_API_KEY]"
      }
    },
    "Obsidian": {
      "command": "/usr/bin/npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/var/home/[YOUR_LINUX_USER]/Obsidian Vaults/[YOUR_VAULT_NAME]"
      ]
    },
    "Mem0": {
      "command": "mem0-mcp-server",
      "args": [],
      "env": {
        "MEM0_API_KEY": "[YOUR_MEM0_API_KEY]",
        "MEM0_DEFAULT_USER_ID": "[YOUR_PREFERRED_USERNAME]"
      }
    }
  },
  "preferences": {
    "menuBarEnabled": true,
    "coworkScheduledTasksEnabled": false,
    "sidebarMode": "chat",
    "legacyQuickEntryEnabled": false
  }
}

```

## Native Integrations (Post-Install)

After opening Claude Desktop, go to **Settings > Connectors** to natively activate:

* Google Drive / Gmail / Calendar
* GitHub

This removes the need for manual API configuration for these services.

## Credits & Tools Used

This tutorial uses and integrates several incredible open-source tools:

* **[Claude Desktop (Linux Build)](https://github.com/aaddrick/claude-desktop-debian):** Unofficial build maintained by the community to bring Claude to Linux.
* **[Model Context Protocol (MCP)](https://modelcontextprotocol.io/):** Anthropic's open standard for connecting AI to data.
* **[Distrobox](https://github.com/89luca89/distrobox):** Essential tool for running any Linux distribution inside your terminal.
* **[Sequential Thinking MCP](https://github.com/modelcontextprotocol/servers):** Step-by-step reasoning server.
* **[Apify MCP](https://github.com/apify/apify-mcp-server):** Official Apify platform integration for web automation and scraping.
* **[Mem0](https://github.com/mem0ai/mem0):** The intelligent memory layer for personalized AI.
* **[Arch Linux](https://archlinux.org/):** The solid, rolling-release base used in the container.

### Important Notes

* **Absolute Paths:** In the JSON file, always use absolute paths (e.g., `/var/home/...` or `/home/...`) to avoid ambiguity between the Host and the Container.
* **Package Names:** Note that the Apify package on NPM is named `@apify/actors-mcp-server`.
* **Obsidian:** We chose to use `@modelcontextprotocol/server-filesystem` to access Obsidian, as it is faster and does not require third-party plugins (unlike `mcp-obsidian`).
