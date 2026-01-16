# Agrisale Official Website

Official download website for Agrisale series - agricultural sales management systems.

## Products

- **Agrisale** - Local-only version, offline-first, data stored on device
- **AgrisaleWS** - Cloud version with team collaboration and cross-device sync

## Structure

```
├── index.html              # Homepage
├── agrisale/index.html     # Agrisale download page
├── agrisalews/index.html   # AgrisaleWS download page
├── api/*/latest.json       # Version API endpoints
└── assets/css/style.css    # Shared styles
```

## Deployment

- **Website**: Cloudflare Pages (auto-deploy from GitHub)
- **Downloads**: Cloudflare R2 at `releases.drflo.org`

See [DEPLOY.md](DEPLOY.md) for detailed instructions.

## Update Version

1. Upload packages to R2: `agrisale-releases/{app}/{version}/`
2. Update HTML files and `api/*/latest.json`
3. Push to GitHub

## License

© 2026 Agrisale. All rights reserved.
