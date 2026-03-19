# Ansible Execution Environment Builder Skill

A comprehensive Claude Code skill for building secure, production-ready Ansible Execution Environments (EE) and Decision Environments (DE) with ansible-builder and ansible-navigator.

## 🎯 Overview

This skill provides an interactive, step-by-step workflow to:
- Build containerized Ansible runtime environments
- Configure collections from Ansible Galaxy and Automation Hub
- Handle authentication securely with environment variables
- Test and validate environments before deployment
- Push images to container registries safely

## ✨ Features

### Core Capabilities
- ✅ Interactive environment type selection (EE vs DE)
- ✅ Automated prerequisite checking and installation
- ✅ Secure token management with environment variables
- ✅ Multiple authentication source support (Automation Hub, private Galaxy)
- ✅ Collection, Python, and system dependency management
- ✅ Build validation and testing with ansible-navigator
- ✅ Container registry push with security scanning

### Security Features
- 🔒 Environment variable token support (AH_TOKEN, GALAXY_TOKEN)
- 🔒 Secure password handling (no command-line exposure)
- 🔒 Build context cleanup
- 🔒 Input validation to prevent command injection
- 🔒 Secret scanning recommendations
- 🔒 Automatic .gitignore generation
- 🔒 Build argument security warnings

## 📦 Installation

### Download and Install

```bash
# Download the skill file
# (or clone this repository)

# Install the skill
claude skill install ansible-ee-builder.skill

# Verify installation
claude skill list | grep ansible-ee-builder
```

### Manual Installation

```bash
# Copy to Claude skills directory
mkdir -p ~/.claude/skills/
cp ansible-ee-builder.skill ~/.claude/skills/
```

## 🚀 Quick Start

### 1. Set Up Prerequisites

The skill will check for and help install:
- **podman** - Container runtime
- **ansible-builder** - EE/DE builder tool
- **ansible-navigator** - Testing and execution tool

```bash
# The skill will run these checks automatically
podman --version
ansible-builder --version
ansible-navigator --version
```

### 2. Set Up Authentication (Optional)

For Red Hat Automation Hub collections:

```bash
# Set token as environment variable (development)
export AH_TOKEN="your_automation_hub_token_here"

# Get your token from:
# https://console.redhat.com/ansible/automation-hub/token
```

For private Galaxy servers:

```bash
export GALAXY_TOKEN="your_galaxy_token_here"
```

### 3. Run the Skill

```bash
# Invoke the skill in Claude Code
/ansible-ee-builder
```

Follow the interactive prompts to:
1. Choose environment type (EE or DE)
2. Configure authentication
3. Select collections and dependencies
4. Build and test the environment
5. Optionally push to a registry

## 📋 Usage Examples

### Example 1: Simple EE with Public Collections

```bash
# No token needed for public Galaxy collections
/ansible-ee-builder

# The skill will guide you to create:
# - ansible.posix
# - community.general
# With Python packages: jmespath, pytz
```

### Example 2: Production EE with Automation Hub

```bash
# Set your Automation Hub token
export AH_TOKEN="eyJhbGci0iJ9..."

# Run the skill
/ansible-ee-builder

# Build with certified collections:
# - ansible.controller
# - redhat.rhel_system_roles
# - amazon.aws (certified)
```

### Example 3: Decision Environment for Event-Driven Ansible

```bash
# Set token
export AH_TOKEN="your_token"

# Run the skill and select "Decision Environment"
/ansible-ee-builder

# Collections:
# - ansible.eda
# - dynatrace.event_driven_ansible
```

## 🔒 Security Best Practices

### Token Management

**✅ DO**:
- Use environment variables for tokens in development
- Use secret management systems in production (Vault, AWS Secrets Manager)
- Rotate tokens regularly (weekly/monthly)
- Unset tokens after use: `unset AH_TOKEN`
- Use short-lived tokens

**❌ DON'T**:
- Use `-p password` flag (exposes in shell history)
- Commit ansible.cfg with tokens to git
- Store tokens in persistent dotfiles for production
- Push images with tokens to public registries

### Build Argument Warning

⚠️ **CRITICAL**: Build arguments (`--build-arg`) are **NOT secure**!

Build args are stored in:
- Image history: `podman history <image>`
- Image metadata: `podman inspect <image>`

Anyone with image access can extract tokens!

**For production**: Use Podman secrets or multi-stage builds instead.

### Recommended Workflow

```bash
# 1. Generate short-lived token
# Visit: https://console.redhat.com/ansible/automation-hub/token

# 2. Set for single session only
export AH_TOKEN="token_here"

# 3. Build your EE
/ansible-ee-builder

# 4. Clean up
unset AH_TOKEN
rm -rf context/

# 5. Scan for secrets (before pushing)
pip install ggshield
podman save my-ee:latest | ggshield secret scan docker-archive -
```

## 🏗️ Architecture

### Directory Structure Created

```
my-ee/
├── .gitignore                    # Prevents committing secrets
├── execution-environment.yml     # Main EE definition
├── ansible-collections.yml       # Collection requirements
├── python-packages.txt          # Python dependencies
├── system-packages.txt          # System dependencies
├── ansible.cfg                  # Galaxy/Hub configuration
└── context/                     # Build context (delete after build)
    └── _build/
        ├── Containerfile
        ├── requirements.yml
        └── configs/
```

### Build Process Flow

1. **Prerequisites Check** → Install missing tools if needed
2. **Authentication** → Check registries and tokens
3. **Requirements Gathering** → Collections, Python, system packages
4. **Configuration Creation** → Generate all config files
5. **Input Validation** → Prevent injection attacks
6. **Build Execution** → ansible-builder creates the image
7. **Post-Build Cleanup** → Delete sensitive build context
8. **Testing** → Validate with ansible-navigator
9. **Security Scanning** → Check for leaked secrets
10. **Registry Push** → Deploy to registry (optional)

## 🔧 Configuration Files

### execution-environment.yml

Defines the base image, dependencies, and build steps:

```yaml
---
version: 3
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel8:latest

dependencies:
  galaxy: ansible-collections.yml
  python: python-packages.txt
  system: system-packages.txt

options:
  package_manager_path: /usr/bin/microdnf

additional_build_steps:
  prepend_base:
    - RUN $PYCMD -m pip install --upgrade pip setuptools
  prepend_galaxy:
    - ADD _build/configs/ansible.cfg /etc/ansible/ansible.cfg
    - ARG AH_TOKEN
    - ENV ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN=$AH_TOKEN
```

### ansible.cfg

Configures collection sources:

```ini
[galaxy]
server_list = automation_hub, release_galaxy

[galaxy_server.automation_hub]
url=https://console.redhat.com/api/automation-hub/content/published/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
# Token provided via AH_TOKEN environment variable

[galaxy_server.release_galaxy]
url=https://galaxy.ansible.com/
```

### .gitignore

Prevents committing sensitive files:

```
# Ansible Builder artifacts
context/
*.log

# Sensitive configuration
ansible.cfg
.env
.ansible_tokens

# Ansible artifacts
*.retry
.ansible/
```

## 🌍 CI/CD Integration

### GitHub Actions

```yaml
name: Build EE
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install ansible-builder
        run: pip install ansible-builder

      - name: Login to Red Hat Registry
        run: |
          echo "${{ secrets.RH_PASSWORD }}" | \
          podman login registry.redhat.io \
            -u "${{ secrets.RH_USERNAME }}" \
            --password-stdin

      - name: Build EE
        env:
          AH_TOKEN: ${{ secrets.AUTOMATION_HUB_TOKEN }}
        run: |
          ansible-builder build \
            --tag quay.io/${{ github.repository_owner }}/my-ee:latest \
            --build-arg AH_TOKEN=${AH_TOKEN} \
            --verbosity 2

      - name: Scan for secrets
        run: |
          pip install ggshield
          podman save my-ee:latest | ggshield secret scan docker-archive -

      - name: Push to Quay
        if: github.ref == 'refs/heads/main'
        run: |
          echo "${{ secrets.QUAY_PASSWORD }}" | \
          podman login quay.io \
            -u "${{ secrets.QUAY_USERNAME }}" \
            --password-stdin
          podman push quay.io/${{ github.repository_owner }}/my-ee:latest
```

### GitLab CI

```yaml
build-ee:
  image: quay.io/ansible/creator-ee:latest
  script:
    - echo "${RH_PASSWORD}" | podman login registry.redhat.io -u "${RH_USERNAME}" --password-stdin
    - ansible-builder build --tag my-ee:latest --build-arg AH_TOKEN=${AH_TOKEN} --verbosity 2
    - podman save my-ee:latest | ggshield secret scan docker-archive -
  variables:
    AH_TOKEN: ${AUTOMATION_HUB_TOKEN}
```

### Jenkins Pipeline

```groovy
pipeline {
  environment {
    AH_TOKEN = credentials('automation-hub-token')
    RH_CREDS = credentials('redhat-registry')
  }
  stages {
    stage('Build EE') {
      steps {
        sh '''
          echo "${RH_CREDS_PSW}" | podman login registry.redhat.io \
            -u "${RH_CREDS_USR}" --password-stdin

          ansible-builder build \
            --tag my-ee:latest \
            --build-arg AH_TOKEN=${AH_TOKEN} \
            --verbosity 2

          rm -rf context/
        '''
      }
    }
  }
}
```

## 🐛 Troubleshooting

### Build fails with "unauthorized" error

```bash
# Verify Red Hat registry authentication
podman login registry.redhat.io

# Check subscription status at:
# https://access.redhat.com/management/subscriptions
```

### Collection installation fails

```bash
# Verify token is set
echo "AH_TOKEN: ${AH_TOKEN:+SET}"

# Test token manually
ansible-galaxy collection install ansible.controller --token ${AH_TOKEN}

# Get fresh token from:
# https://console.redhat.com/ansible/automation-hub/token
```

### Token appears in image history

```bash
# Check image history
podman history my-ee:latest

# If token is visible, DELETE the image immediately
podman rmi my-ee:latest

# Rebuild using secrets instead of build-args (Podman 4.0+)
```

### Large image size

```bash
# Use minimal base images
# Change: ee-supported-rhel8 → ee-minimal-rhel8

# Install only required collections
# Don't install entire namespaces

# Combine RUN commands to reduce layers
```

## 📚 Resources

- **Ansible Builder Documentation**: https://ansible.readthedocs.io/projects/builder/
- **Ansible Navigator Documentation**: https://ansible.readthedocs.io/projects/navigator/
- **Red Hat Automation Hub**: https://console.redhat.com/ansible/automation-hub
- **Ansible Galaxy**: https://galaxy.ansible.com/
- **Execution Environment Guide**: https://docs.ansible.com/automation-controller/latest/html/userguide/execution_environments.html

## 🤝 Contributing

Contributions are welcome! Areas for improvement:
- Multi-stage build examples
- Podman secrets integration
- Additional security scanners
- More CI/CD platform examples
- Enhanced error handling

## 📄 License

This skill is provided as-is for use with Claude Code.

## 🔄 Changelog

### Version 2.0.0 (2026-03-19)
- ✨ Added environment variable token support (AH_TOKEN, GALAXY_TOKEN)
- 🔒 Enhanced security features and warnings
- 🔒 Secure password handling (password-stdin)
- 🔒 Build context cleanup automation
- 🔒 Input validation for command injection prevention
- 🔒 Automatic .gitignore generation
- 🔒 Secret scanning integration
- 📚 Comprehensive security best practices
- 🤖 Automated prerequisite installation
- 📖 CI/CD integration examples

### Version 1.0.0
- Initial release
- Basic EE/DE building workflow
- Collection and dependency management

## ⚠️ Security Disclosure

If you discover a security vulnerability in this skill, please report it to the repository maintainer. Do not create public issues for security vulnerabilities.

## 🙏 Acknowledgments

Built for the Ansible automation community. Special thanks to:
- Red Hat Ansible Team for ansible-builder and ansible-navigator
- Claude Code team for the skills framework
- The Ansible community for feedback and contributions

---

**Made with ❤️ for secure Ansible automation**
