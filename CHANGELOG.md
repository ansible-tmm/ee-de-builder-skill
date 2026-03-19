# Changelog

All notable changes to the ansible-ee-builder skill will be documented in this file.

## [2.0.0] - 2026-03-19

### 🔒 Security Enhancements (CRITICAL)

#### Added
- **Environment variable token support**: Primary authentication method using `AH_TOKEN` and `GALAXY_TOKEN`
- **Secure password handling**: Replaced `-p password` flags with interactive prompts and `--password-stdin`
- **Build argument security warnings**: Prominent warnings about token exposure in image metadata
- **Build context cleanup**: Automatic cleanup of sensitive `context/` directory after builds
- **Input validation**: Prevents command injection attacks on image names and parameters
- **Automatic .gitignore generation**: Prevents committing sensitive files to version control
- **Secret scanning integration**: Recommends scanning images with ggshield/trufflehog before pushing
- **Token lifecycle guidance**: Best practices for token generation, usage, rotation, and cleanup
- **Environment variable security warnings**: Explains risks of persisting tokens in dotfiles

#### Changed
- **Registry authentication**: Now uses secure methods (interactive or password-stdin)
- **Token priority**: ENV vars → ansible.cfg → user prompt
- **Best practices section**: Comprehensive security-first approach with DO/DON'T lists
- **Prerequisite installation**: Now asks for permission before installing tools
- **Step numbering**: Updated to include new security steps (14→15 for registry push)

#### Security Fixes
- Fixed: Passwords exposed in shell history via `-p` flag
- Fixed: Passwords exposed in process list via `-p` flag
- Fixed: No cleanup of build context containing sensitive data
- Fixed: No input validation allowing command injection
- Fixed: No guidance on token rotation or expiration
- Fixed: Environment variables persisted unsafely in dotfiles
- Fixed: No warnings about build-arg token exposure

### ✨ Features

#### Added
- **Multi-token support**: Can use both AH_TOKEN and GALAXY_TOKEN simultaneously
- **CI/CD integration examples**: GitHub Actions, GitLab CI, Jenkins Pipeline
- **Automated prerequisite checking**: Detects and offers to install missing tools
- **Platform-specific installation**: Auto-detects OS (dnf/apt/brew) for podman installation
- **Post-build verification**: Step to verify pushed images work correctly
- **Security scanning step**: Optional but recommended secret scanning before push

### 📚 Documentation

#### Added
- **Comprehensive README.md**: Full usage guide, examples, and troubleshooting
- **SECURITY_RECOMMENDATIONS.md**: Detailed security analysis and fixes
- **CHANGELOG.md**: Version history and changes
- **Quick start guide**: Fast path to building first EE
- **Architecture section**: Explains directory structure and build flow
- **Configuration file templates**: Documented examples for all config files
- **Troubleshooting guide**: Common issues and solutions

### 🔧 Improvements

#### Changed
- **ansible.cfg templates**: Now includes both token and tokenless variants
- **execution-environment.yml**: Support for multiple build arguments
- **Build command options**: Three clear options (env vars, direct arg, no auth)
- **Token setting instructions**: Temporary vs persistent with security trade-offs
- **File organization**: Improved step-by-step flow with security checkpoints

### 📦 Distribution

#### Added
- **Packaged .skill file**: 8.0 KB compressed (was 5.8 KB)
- **SHA256 checksum**: For integrity verification
- **Installation instructions**: Multiple installation methods
- **Version information**: Semantic versioning

### Breaking Changes

None. This version maintains backward compatibility while adding security improvements.

### Migration Guide

If upgrading from 1.x:

1. **Update token handling**:
   ```bash
   # Old (INSECURE)
   podman login registry.redhat.io -u user -p password

   # New (SECURE)
   podman login registry.redhat.io -u user
   # Or: echo "$PASSWORD" | podman login registry.redhat.io -u user --password-stdin
   ```

2. **Use environment variables**:
   ```bash
   # Set token
   export AH_TOKEN="your_token"

   # Build
   /ansible-ee-builder

   # Cleanup
   unset AH_TOKEN
   ```

3. **Add cleanup step**:
   ```bash
   # After building
   rm -rf context/
   ```

4. **Create .gitignore**:
   ```bash
   # Skill now auto-creates .gitignore
   # If upgrading existing projects, add manually
   echo "context/" >> .gitignore
   echo "ansible.cfg" >> .gitignore
   ```

### Known Issues

1. **Build args still used**: While we warn about the security implications, build-args remain the primary method in ansible-builder. Podman secrets require manual Containerfile modification.

   **Workaround**: Use short-lived tokens and delete images after testing.

2. **No token validation**: Skill doesn't verify token validity before build.

   **Workaround**: Test token manually with `ansible-galaxy collection list`

3. **Platform detection**: Auto-installation may not work on all Linux distributions.

   **Workaround**: Install prerequisites manually if auto-install fails.

### Deprecation Notices

None in this version.

### Future Roadmap

- [ ] Podman secrets integration (requires Podman 4.0+)
- [ ] Multi-stage build templates
- [ ] Token validation before build
- [ ] Automatic vulnerability scanning with trivy
- [ ] Support for custom base images
- [ ] Air-gapped installation support
- [ ] Collection dependency resolution
- [ ] Image size optimization automation

## [1.0.0] - Initial Release

### Features
- Basic EE/DE building workflow
- Collection and dependency management
- Registry authentication
- ansible-navigator testing
- Registry push functionality

---

**Legend**:
- 🔒 Security
- ✨ Features
- 📚 Documentation
- 🔧 Improvements
- 🐛 Bug Fixes
- 📦 Distribution
- ⚠️ Breaking Changes
