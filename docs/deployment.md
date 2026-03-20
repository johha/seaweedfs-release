# Deployment Guide

> **⚠️ POC Status**: This is a proof-of-concept. Test thoroughly before using in any environment.

This guide walks through deploying the SeaweedFS BOSH release.

## Prerequisites

- BOSH Director deployed and configured
- BOSH CLI v7.0.0 or later
- Cloud config with appropriate networks, VM types, and disk types
- (Optional) Cloud Foundry deployment for integration testing

## Step 1: Upload Required Releases

```bash
# Upload BPM release (required dependency)
bosh upload-release --sha1 beb0ab09dd6e0935c87f8efee047bd5a4a29ece2 \
  https://bosh.io/d/github.com/cloudfoundry/bpm-release?v=1.2.15

# Upload SeaweedFS release
bosh upload-release https://github.com/YOUR_ORG/seaweedfs-release/releases/download/v1.0.0/seaweedfs-release-1.0.0.tgz
```

## Step 2: Configure Variables

Create a `vars.yml` file with your environment-specific values:

```yaml
seaweedfs_domain_name: blobstore.system.example.com
```

Or use the example template:

```bash
cp examples/vars-example.yml vars.yml
# Edit vars.yml with your values
```

## Step 3: Deploy Standalone SeaweedFS

```bash
bosh -d seaweedfs deploy examples/seaweedfs-standalone.yml \
  --vars-file=vars.yml
```

This will deploy a single-instance SeaweedFS with:
- 100GB persistent disk
- S3 API on port 8333 (HTTPS)
- Auto-generated credentials
- BBR backup enabled

## Step 4: Verify Deployment

Check instance status:

```bash
bosh -d seaweedfs instances
bosh -d seaweedfs ssh seaweedfs/0
```

Inside the VM, check processes:

```bash
sudo su -
bpm list
monit summary
```

Check logs:

```bash
tail -f /var/vcap/sys/log/seaweedfs/*.log
```

## Step 5: Test S3 API

Retrieve credentials:

```bash
bosh -d seaweedfs manifest | grep seaweedfs_access_key -A 2
```

Configure AWS CLI:

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_ENDPOINT_URL="https://blobstore.system.example.com:8333"
```

Test operations:

```bash
# List buckets
aws s3 ls --endpoint-url="${AWS_ENDPOINT_URL}"

# Upload a test file
echo "test" > test.txt
aws s3 cp test.txt s3://packages/test.txt --endpoint-url="${AWS_ENDPOINT_URL}"

# List objects
aws s3 ls s3://packages/ --endpoint-url="${AWS_ENDPOINT_URL}"

# Download
aws s3 cp s3://packages/test.txt downloaded.txt --endpoint-url="${AWS_ENDPOINT_URL}"
```

## Integrating with Cloud Foundry

### Option 1: New CF Deployment

```bash
bosh -d cf deploy cf-deployment.yml \
  -o examples/operations/use-seaweedfs-blobstore.yml \
  --vars-file=cf-vars.yml
```

### Option 2: Update Existing CF

**⚠️ WARNING**: This will replace your existing blobstore. Backup first!

```bash
# Backup current deployment
bosh -d cf manifest > cf-backup.yml

# Apply SeaweedFS ops file
bosh -d cf deploy cf-deployment.yml \
  -o examples/operations/use-seaweedfs-blobstore.yml \
  --vars-file=cf-vars.yml
```

### Verify CF Integration

```bash
# Check that Cloud Controller can reach blobstore
bosh -d cf ssh api/0

# Inside api VM
curl -k https://blobstore.system.example.com:8333/
```

Push a test app:

```bash
cf push test-app
```

## Backup and Restore

### Backup

```bash
# Install bbr CLI
wget https://github.com/cloudfoundry-incubator/bosh-backup-and-restore/releases/latest/download/bbr-linux-amd64
chmod +x bbr-linux-amd64
sudo mv bbr-linux-amd64 /usr/local/bin/bbr

# Perform backup
bbr deployment \
  --target $(bosh env --json | jq -r '.Tables[0].Rows[0].url') \
  --username admin \
  --password $(bosh int <(bosh int ~/workspace/bosh-deployment/creds.yml --path /admin_password)) \
  --deployment seaweedfs \
  --ca-cert <(bosh int ~/workspace/bosh-deployment/creds.yml --path /director_ssl/ca) \
  backup
```

### Restore

```bash
bbr deployment \
  --target $(bosh env --json | jq -r '.Tables[0].Rows[0].url') \
  --username admin \
  --password $(bosh int <(bosh int ~/workspace/bosh-deployment/creds.yml --path /admin_password)) \
  --deployment seaweedfs \
  --ca-cert <(bosh int ~/workspace/bosh-deployment/creds.yml --path /director_ssl/ca) \
  restore --artifact-path seaweedfs_TIMESTAMP
```

## Scaling and Performance

### Disk Sizing

Default: 100GB persistent disk

To increase:

```yaml
instance_groups:
- name: seaweedfs
  persistent_disk_type: 500GB  # or larger
```

### Volume Limits

Adjust max volumes based on disk size:

```yaml
properties:
  seaweedfs:
    volume:
      max_volumes: 200  # Increase for larger disks
```

Rule of thumb: max_volumes * volume_size_limit_mb ≈ disk_size

### Performance Tuning

For better performance:

1. Use faster disk types (SSD)
2. Increase VM resources (more CPU/RAM)
3. Tune volume size limits
4. Monitor using SeaweedFS metrics

## Monitoring

### Health Checks

```bash
# Master status
curl http://SEAWEEDFS_IP:9333/cluster/status

# Volume status
curl http://SEAWEEDFS_IP:8080/status

# Filer status
curl http://SEAWEEDFS_IP:8888/
```

### Logs

```bash
# View all logs
bosh -d seaweedfs logs

# Follow specific component
bosh -d seaweedfs ssh seaweedfs/0
tail -f /var/vcap/sys/log/seaweedfs/s3.stdout.log
```

## Troubleshooting

See [troubleshooting.md](troubleshooting.md) for common issues and solutions.

## Uninstalling

```bash
# Delete deployment
bosh -d seaweedfs delete-deployment

# Remove orphaned disks if needed
bosh clean-up --all
```

## Next Steps

- Set up monitoring and alerting
- Configure log forwarding
- Test backup/restore procedures
- Load test with realistic workloads
- Document operational runbooks
