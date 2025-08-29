# Rov's Tech Blog

A comprehensive technical blog covering home lab projects, Microsoft cloud technologies, security implementations, and development best practices.

## Quick Start

### Prerequisites
- Python 3.8+
- pip (Python package manager)

### Installation
```bash
# Install MkDocs and required plugins
pip install mkdocs-material

# Clone the repository
git clone https://github.com/RovxBot/rovblog.git
cd rovblog

# Serve locally
mkdocs serve
```

Visit `http://127.0.0.1:8000` to view the blog locally.

### Windows Users
Simply run `serve.bat` to start the development server.

## Content Structure

```
docs/
├── homelab/                 # Home lab projects
│   ├── K8s/                # Kubernetes setup journey
│   └── Dokploy/            # Docker Swarm with Dokploy
├── WDAC/                   # Windows Defender Application Control
├── Conditional Acess/      # Conditional Access policies
├── PIM/                    # Privileged Identity Management
├── Version Control/        # DevOps and version control
├── Tailscale/             # Networking with Tailscale
├── Game/                  # Creative writing and game lore
└── assets/                # Images and media files
```

## Features

- **Material Design**: Clean, responsive theme
- **Search**: Full-text search across all content
- **Code Highlighting**: Syntax highlighting for multiple languages
- **Admonitions**: Info boxes, warnings, tips, and more
- **Navigation**: Tabbed navigation with sections
- **Mobile Friendly**: Responsive design for all devices
- **Dark/Light Mode**: Automatic theme switching

## Writing Content

All content is written in Markdown with additional features:

### Admonitions
```markdown
!!! tip "Pro Tip"
    This is a helpful tip for readers.

!!! warning "Important"
    This is something to be careful about.
```

### Code Blocks
```markdown
```bash
# This is a bash command
echo "Hello World"
```
```

### Links and Navigation
All internal links are automatically validated during build. Use relative paths from the docs directory.

## Deployment

The site is automatically deployed to GitHub Pages when changes are pushed to the main branch.

### Manual Deployment
```bash
# Build the site
mkdocs build

# Deploy to GitHub Pages
mkdocs gh-deploy
```

## Content Categories

- **Home Lab**: Kubernetes, Docker Swarm, hardware setup
- **Security**: WDAC, Conditional Access, PIM
- **DevOps**: Version control, Azure DevOps, CI/CD
- **Networking**: Tailscale, VPN configurations
- **Creative**: Game lore and world building

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add your content in the appropriate directory
4. Update the navigation in `mkdocs.yml`
5. Test locally with `mkdocs serve`
6. Submit a pull request

## Contact

- **GitHub**: [@RovxBot](https://github.com/rovxbot)
- **Discord**: [Rov#1234](https://discord.com/users/253028999666728960)
- **Last.fm**: [rovbot](https://www.last.fm/user/rovbot)
- **Steam**: [rovbot](https://steamcommunity.com/id/rovbot/)

## License

This project is open source and available under the [MIT License](LICENSE).
