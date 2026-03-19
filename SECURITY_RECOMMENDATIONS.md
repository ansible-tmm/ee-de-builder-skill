# Security Recommendations for ansible-ee-builder Skill

## Critical Fixes Needed

### 1. Password Handling
**Current (UNSAFE)**:
```bash
podman login registry.redhat.io -u <username> -p <password>
```

**Recommended**:
```bash
# Interactive prompt (most secure)
podman login registry.redhat.io -u <username>

# Or with stdin (for automation)
echo "${PASSWORD}" | podman login registry.redhat.io -u <username> --password-stdin
```

### 2. Build Argument Token Exposure
**Current (UNSAFE)**: Tokens in build args are stored in image history

**Recommended**: Add prominent warning in skill:

```
⚠️  SECURITY WARNING: Build arguments are stored in image metadata!

Build args (--build-arg AH_TOKEN=xxx) will be visible in:
- Image history: podman history <image>
- Image inspection: podman inspect <image>
- Docker Hub / Quay.io if image is published

NEVER use build args for sensitive tokens in production!

Recommended alternatives:
1. Use Podman/BuildKit secrets (Podman 4.0+)
2. Mount tokens at runtime only
3. Use short-lived tokens that expire quickly
```

**Better Implementation**:
```yaml
# In execution-environment.yml
additional_build_steps:
  prepend_galaxy:
    - RUN --mount=type=secret,id=ah_token \
        ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN=$(cat /run/secrets/ah_token) \
        ansible-galaxy collection install -r requirements.yml
```

Then build with:
```bash
echo "${AH_TOKEN}" | podman build \
  --secret id=ah_token,src=/dev/stdin \
  -f context/Containerfile \
  -t my-ee:latest context
```

### 3. Shell History Exposure
**Add to skill**: Always warn about shell history

```
⚠️  SECURITY TIP: Clear sensitive commands from history

# Prefix commands with space to avoid history (if HISTCONTROL=ignorespace)
 export AH_TOKEN="secret_token"

# Or disable history temporarily
set +o history
export AH_TOKEN="secret_token"
set -o history

# Or clear specific entries
history -d $(history | grep AH_TOKEN | awk '{print $1}')
```

### 4. ansible.cfg Token Storage
**Add warning**:
```
⚠️  NEVER commit ansible.cfg with tokens to version control!

Add to .gitignore:
ansible.cfg
*.cfg
context/

Use .env files instead (also in .gitignore):
# .env
AH_TOKEN=your_token_here
```

## Additional Security Enhancements

### 5. Input Validation
Add to skill instructions:

```bash
# Validate image names (alphanumeric, dashes, slashes only)
if [[ ! "$IMAGE_NAME" =~ ^[a-zA-Z0-9_/\.\-:]+$ ]]; then
    echo "Error: Invalid image name"
    exit 1
fi

# Validate collection names
if [[ ! "$COLLECTION_NAME" =~ ^[a-zA-Z0-9_\.]+$ ]]; then
    echo "Error: Invalid collection name"
    exit 1
fi
```

### 6. Build Context Cleanup
Add mandatory cleanup step:

```bash
# After successful build, clean up build context
rm -rf context/

# Or if you need to inspect it
echo "⚠️  Build context contains sensitive data!"
echo "Review context/ directory and delete when done:"
echo "  rm -rf context/"
```

### 7. Token Best Practices Section
Add to skill:

```markdown
## Token Security Best Practices

1. **Use short-lived tokens**: Refresh Automation Hub tokens regularly
2. **Principle of least privilege**: Use tokens with minimal required permissions
3. **Separate tokens per environment**: Dev, staging, prod should have different tokens
4. **Rotate tokens**: If a token might be compromised, rotate immediately
5. **Audit token usage**: Review Red Hat console for unexpected token usage
6. **Use secret managers**: In production, use Vault, AWS Secrets Manager, etc.
7. **Never log tokens**: Ensure CI/CD doesn't log token values
8. **Scan images**: Before pushing, scan for leaked secrets with tools like:
   - `ggshield` (GitGuardian)
   - `gitleaks`
   - `trufflehog`
```

### 8. Environment Variable Security
Add warning about environment variables:

```markdown
## Environment Variable Security Considerations

⚠️  Environment variables are NOT fully secure:
- Visible to all processes started from the same shell
- May appear in `/proc/<pid>/environ`
- Can leak in error messages and logs
- Inherited by child processes

For maximum security:
1. Set only when needed, unset immediately after
2. Use process-specific environments (not global)
3. In production, use secret management systems
4. Never echo/print the actual token value
```

### 9. Image Scanning
Add post-build security check:

```bash
# Scan for secrets before pushing
echo "🔍 Scanning image for leaked secrets..."
if command -v ggshield &> /dev/null; then
    podman save localhost/test-ee:latest | ggshield secret scan docker-archive -
else
    echo "⚠️  Install ggshield to scan for leaked secrets: pip install ggshield"
fi
```

### 10. Multi-Stage Build for Secrets
Recommend multi-stage builds to avoid token leakage:

```dockerfile
# Stage 1: Build with secrets
FROM base AS builder
ARG AH_TOKEN
RUN ansible-galaxy collection install ...

# Stage 2: Copy only artifacts (no token in final image)
FROM base
COPY --from=builder /usr/share/ansible/collections /usr/share/ansible/collections
# Token is NOT in final image!
```

## Summary of Required Changes

1. ✅ Add prominent security warnings
2. ✅ Remove password from command line examples
3. ✅ Document build-arg security implications
4. ✅ Add input validation examples
5. ✅ Add build context cleanup step
6. ✅ Add .gitignore recommendations
7. ✅ Add token rotation/expiration guidance
8. ✅ Add secret scanning recommendation
9. ✅ Document multi-stage build approach
10. ✅ Add shell history security tips
