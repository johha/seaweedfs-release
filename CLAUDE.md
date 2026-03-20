# CLAUDE.md - AI Assistant Context

> **⚠️ Dev/Test Only**: This is a BOSH release for **development and test environments**. High availability is **out of scope by design**. Focus on simplicity for local dev, not production hardening.

## Project Overview

This is a BOSH release for SeaweedFS, providing an S3-compatible blobstore for Cloud Foundry **development and test environments**. It is designed as a proof-of-concept with a deliberately simple single-VM architecture, perfect for local development, CI/CD testing, and evaluation.

**This is NOT a production-ready release.** For production, users should use AWS S3, GCS, Azure Blob, or other proven solutions.

## Project Structure

This is a standard BOSH release with the following key components:

- **jobs/seaweedfs/**: The main BOSH job containing templates and configuration
  - `spec`: Job specification with properties and package dependencies
  - `monit`: Process monitoring configuration
  - `templates/`: ERB templates for configuration and control scripts
    - `bpm.yml.erb`: BPM process configuration
    - `pre-start.sh.erb`, `post-start.sh.erb`, `drain.sh.erb`: Lifecycle scripts
    - `config/`: Configuration files for SeaweedFS components
    - `bin/backup.erb`, `bin/restore.erb`: BBR integration scripts

- **packages/seaweedfs/**: Package definition for SeaweedFS binary
  - `spec`: Package specification and blob references
  - `packaging`: Script to download and install SeaweedFS

- **config/**: Release-level configuration
  - `final.yml`: Final release configuration (blobstore, etc.)
  - `blobs.yml`: References to binary blobs

- **examples/**: Example deployment manifests and operations files

## Architecture

SeaweedFS runs as a single-VM deployment with four processes:
1. Master server (metadata management)
2. Volume server (data storage)
3. Filer (file system abstraction)
4. S3 gateway (S3-compatible API)

All processes are managed by BPM and run on the same VM with a persistent disk.

## Key Design Decisions

1. **Single-VM Design**: For simplicity - perfect for dev/test, intentionally not HA
2. **No Replication**: Single-node setup for dev/test use cases (HA is out of scope)
3. **BPM Process Management**: Following BOSH best practices
4. **BBR Integration**: Backup/restore support for data safety
5. **TLS Support**: Optional TLS for S3 API endpoint
6. **S3 API Only**: Focus on S3 compatibility (not WebDAV like singleton-blobstore)
7. **Auto-generated Credentials**: BOSH handles all credential management (no manual setup)
8. **mTLS Support**: Mutual TLS for secure CF service communication (matches CF security standards)

**Important**: Multi-node HA is **deliberately out of scope**. This release is for dev/test environments only.

## Development Guidelines

### Code Style
- Use 2-space indentation for YAML and Ruby
- ERB templates should use `<% %>` (not `<%= %>`) unless outputting values
- Follow BOSH release conventions and best practices
- Keep configuration templates readable and well-commented

### Property Naming
- Use nested properties: `seaweedfs.s3.port` not `seaweedfs_s3_port`
- Provide sensible defaults for all optional properties
- Document all properties with clear descriptions

### Process Management
- All processes must be managed by BPM
- Use proper signal handling for graceful shutdown
- Implement health checks in post-start script
- Use drain script for coordinated shutdown

### Security
- Never commit credentials or secrets
- Use BOSH-generated credentials (access_key, secret_key, certificates)
- All credentials auto-generated via variables section in manifests
- Support TLS and mTLS for external endpoints
- Follow principle of least privilege
- See docs/security.md for detailed security guidelines

### Testing
- Test manually with `bosh create-release --force` before committing
- Validate that all templates render correctly
- Test backup/restore functionality
- Verify S3 API compatibility with standard tools (aws-cli, s3cmd)

## BOSH Release Workflow

### Creating a Dev Release
```bash
bosh create-release --force --tarball=/tmp/seaweedfs-dev.tgz
```

### Uploading a Release
```bash
bosh upload-release /tmp/seaweedfs-dev.tgz
```

### Deploying
```bash
bosh -d seaweedfs deploy examples/singleton-seaweedfs.yml \
  --vars-file=vars.yml
```

### Creating a Final Release
```bash
bosh create-release --final --version=x.y.z
git add .
git commit -m "Release version x.y.z"
git tag vx.y.z
git push && git push --tags
```

## Common Tasks

### Updating SeaweedFS Version
1. Update the blob reference in `packages/seaweedfs/spec`
2. Download the new binary: `bosh add-blob path/to/weed seaweedfs/weed-x.y.z`
3. Update version references in documentation
4. Test the new version thoroughly

### Adding a New Property
1. Add property definition to `jobs/seaweedfs/spec`
2. Update relevant template to use the property
3. Update example manifests
4. Document the property in README.md

### Debugging
- Check BPM logs: `/var/vcap/sys/log/seaweedfs/`
- Check monit status: `monit summary`
- Check process status: `ps aux | grep weed`
- Check BPM processes: `bpm list`

## Dependencies

### External Dependencies
- **SeaweedFS**: Binary downloaded from GitHub releases
- **BPM**: BOSH Process Manager (co-located release)

### BOSH CLI Tools
- `bosh-cli`: For release management
- `bosh`: For deployment

## Automation

### Renovate
Renovate is configured to automatically track SeaweedFS releases and create PRs for version updates. Configuration in `.github/renovate.json`.

### GitHub Actions
- **Release Workflow**: Automatically creates GitHub releases when tags are pushed
- **Test Workflow**: Validates that releases can be built successfully

## Integration with Cloud Foundry

This release is designed to work with cf-deployment. An operations file is provided to replace the singleton-blobstore:

```bash
bosh -d cf deploy cf-deployment.yml \
  -o operations/use-seaweedfs-blobstore.yml \
  --vars-file=vars.yml
```

## Reference Material

- [SeaweedFS Documentation](https://github.com/seaweedfs/seaweedfs/wiki)
- [BOSH Documentation](https://bosh.io/docs/)
- [BPM Documentation](https://github.com/cloudfoundry/bpm-release)
- [capi-release blobstore job](https://github.com/cloudfoundry/capi-release/tree/main/jobs/blobstore)

## Troubleshooting

### Common Issues

1. **S3 API not responding**: Check that filer process is running and healthy
2. **Permission denied errors**: Verify ownership of `/var/vcap/store/seaweedfs`
3. **Backup fails**: Ensure BBR job is properly configured and processes can be locked
4. **Volume server out of space**: Check persistent disk size and volume count

### Getting Help

- Check logs in `/var/vcap/sys/log/seaweedfs/`
- Review IMPLEMENTATION_PLAN.md for architecture details
- Consult SeaweedFS documentation for component-specific issues

## Future Enhancements

**Note**: These are out of scope for this dev/test release. If you need these, use production solutions.

See IMPLEMENTATION_PLAN.md for potential future features that are currently out of scope (multi-node HA, replication, etc.).

**Do not implement production features** without explicit user request, as this would violate the dev/test scope.
