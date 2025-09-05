# Beanstalkd COPR Build Repository

Automated COPR build configuration for [Beanstalkd](https://github.com/beanstalkd/beanstalkd) - a simple, fast work queue service.

## Overview

This repository contains the COPR build configuration to automatically build RPM packages for Beanstalkd from the latest git commits on the master branch.

## Features

- **Automated Builds**: Builds directly from the latest Beanstalkd git master branch
- **Smart Versioning**: Automatically increments release numbers for the same git commit
- **Systemd Integration**: Includes proper systemd service configuration
- **SELinux Compatible**: Properly handles SELinux contexts for binlog directories

## COPR Repository

The built packages are available in COPR at:
```
https://copr.fedorainfracloud.org/coprs/YOUR_USERNAME/beanstalkd/
```

### Installation

To use packages from this COPR repository:

```bash
# Enable the COPR repository
sudo dnf copr enable YOUR_USERNAME/beanstalkd

# Install beanstalkd
sudo dnf install beanstalkd

# Start and enable the service
sudo systemctl enable --now beanstalkd
```

## Configuration

The default configuration file is located at `/etc/sysconfig/beanstalkd`. Key settings:

- **ADDR**: Listen address (default: `-l 0.0.0.0`)
- **PORT**: Listen port (default: `-p 11300`)
- **USER**: Run as user (default: `-u beanstalkd`)
- **BINLOG_DIR**: Optional persistent storage directory

## Build Process

The build process:
1. Clones the latest Beanstalkd master branch
2. Detects the version from git tags
3. Checks COPR for existing builds to determine release number
4. Creates source tarball and auxiliary files
5. Generates RPM spec file dynamically
6. Builds SRPM for COPR

## Local Testing

To test the build locally:
```bash
cd .copr
make srpm
```

## Files

- `.copr/Makefile` - COPR build configuration
- `README.md` - This file

## License

The build configuration is provided as-is. Beanstalkd itself is licensed under the MIT License.

## Upstream

- Beanstalkd GitHub: https://github.com/beanstalkd/beanstalkd
- Beanstalkd Website: https://beanstalkd.github.io/

## Contributing

Issues and pull requests can be submitted on GitHub.