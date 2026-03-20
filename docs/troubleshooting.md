# Troubleshooting Guide

> **⚠️ POC Status**: This guide covers common issues in the POC. Additional edge cases may exist.

## Common Issues

### 1. Deployment Fails - Package Compilation

**Symptom:**
```
Error: Compiling package 'seaweedfs': Command 'tar -xzf seaweedfs/weed-3.73-linux-amd64.tar.gz' failed
```

**Solution:**
The SeaweedFS binary blob hasn't been uploaded yet. This POC assumes you'll add the blob:

```bash
# Download SeaweedFS binary
wget https://github.com/seaweedfs/seaweedfs/releases/download/3.73/linux_amd64_full.tar.gz

# Extract and prepare
tar -xzf linux_amd64_full.tar.gz
tar -czf weed-3.73-linux-amd64.tar.gz weed

# Add to BOSH blobstore
bosh add-blob weed-3.73-linux-amd64.tar.gz seaweedfs/weed-3.73-linux-amd64.tar.gz
bosh upload-blobs
```

### 2. S3 API Returns 403 Forbidden

**Symptom:**
```
aws s3 ls --endpoint-url=https://... returns AccessDenied
```

**Possible Causes:**

1. **Wrong credentials**
   ```bash
   # Verify credentials from deployment
   bosh -d seaweedfs manifest | grep -A 5 seaweedfs_access_key
   ```

2. **TLS certificate issues**
   ```bash
   # Test without TLS verification
   aws s3 ls --endpoint-url=https://... --no-verify-ssl

   # If this works, your TLS cert is the issue
   ```

3. **S3 config not loaded**
   ```bash
   bosh -d seaweedfs ssh seaweedfs/0
   cat /var/vcap/jobs/seaweedfs/config/s3.json
   # Verify credentials are present
   ```

### 3. Processes Not Starting

**Symptom:**
```
monit summary shows processes not running
```

**Diagnosis:**

```bash
# Check BPM status
bpm list

# Check logs
tail -f /var/vcap/sys/log/seaweedfs/master.stderr.log
tail -f /var/vcap/sys/log/seaweedfs/volume.stderr.log
tail -f /var/vcap/sys/log/seaweedfs/filer.stderr.log
tail -f /var/vcap/sys/log/seaweedfs/s3.stderr.log
```

**Common Issues:**

1. **Port conflicts**
   - Verify no other process is using ports 9333, 8080, 8888, 8333
   - `netstat -tulpn | grep -E '9333|8080|8888|8333'`

2. **Disk permissions**
   ```bash
   ls -la /var/vcap/store/seaweedfs/
   # Should be owned by vcap:vcap
   chown -R vcap:vcap /var/vcap/store/seaweedfs/
   ```

3. **Insufficient disk space**
   ```bash
   df -h /var/vcap/store
   ```

### 4. Post-Start Script Fails

**Symptom:**
```
Error: 'post-start' script failed with error 1
```

**Check health check logs:**

```bash
/var/vcap/sys/log/seaweedfs/post-start.stdout.log
/var/vcap/sys/log/seaweedfs/post-start.stderr.log
```

**Common Issues:**

1. **Processes not ready in time**
   - Increase MAX_RETRIES in post-start.sh.erb
   - Check if processes are actually starting

2. **Bucket creation fails**
   - May be due to S3 auth issues
   - Check s3.json config is correct

### 5. Volume Server Out of Space

**Symptom:**
```
No writable volumes available
```

**Solutions:**

1. **Increase persistent disk size**
   ```yaml
   instance_groups:
   - name: seaweedfs
     persistent_disk_type: 200GB
   ```

2. **Increase max volumes**
   ```yaml
   properties:
     seaweedfs:
       volume:
         max_volumes: 200
   ```

3. **Clean up old data**
   ```bash
   # Use SeaweedFS commands to compact/garbage collect
   /var/vcap/packages/seaweedfs/bin/weed shell
   > volume.balance -force
   ```

### 6. BBR Backup Fails

**Symptom:**
```
Backup failed: exit status 1
```

**Check backup logs:**

```bash
/var/vcap/sys/log/seaweedfs/backup.stdout.log
/var/vcap/sys/log/seaweedfs/backup.stderr.log
```

**Common Issues:**

1. **Insufficient disk space for backup**
   ```bash
   df -h /var/vcap/store
   ```

2. **Processes can't be stopped**
   - Check if BPM is working: `bpm list`
   - Verify monit is running: `monit summary`

3. **Permission issues**
   ```bash
   ls -la /var/vcap/store/bbr/
   ```

### 7. Restore Fails

**Symptom:**
```
Restore failed: metadata.json not found
```

**Solutions:**

1. **Verify backup artifact structure**
   ```bash
   ls -la seaweedfs_TIMESTAMP/
   # Should contain: metadata.json, master_*.tar.gz, volume_*.tar.gz, etc.
   ```

2. **Check backup metadata**
   ```bash
   cat seaweedfs_TIMESTAMP/metadata.json
   ```

### 8. TLS Certificate Issues

**Symptom:**
```
SSL certificate problem: self signed certificate
```

**Solutions:**

1. **For testing, disable TLS verification:**
   ```bash
   aws s3 ls --endpoint-url=https://... --no-verify-ssl
   ```

2. **Add CA cert to trust store:**
   ```bash
   # Extract CA from deployment
   bosh -d seaweedfs manifest | grep -A 20 seaweedfs_ca

   # Add to system trust store (Ubuntu)
   sudo cp ca.crt /usr/local/share/ca-certificates/
   sudo update-ca-certificates
   ```

3. **Use proper certificates:**
   - Generate certs with correct SAN/CN
   - Use Let's Encrypt for public domains
   - Use internal CA for private domains

### 9. Cloud Foundry Can't Connect

**Symptom:**
```
CF push fails with blobstore errors
```

**Diagnosis:**

1. **Network connectivity**
   ```bash
   bosh -d cf ssh api/0
   curl -k https://blobstore.system.domain:8333/
   ```

2. **DNS resolution**
   ```bash
   dig blobstore.system.domain
   nslookup blobstore.system.domain
   ```

3. **Check CC configuration**
   ```bash
   bosh -d cf manifest | grep -A 20 fog_connection
   ```

4. **Verify credentials match**
   ```bash
   # Compare SeaweedFS credentials
   bosh -d seaweedfs manifest | grep seaweedfs_access_key

   # With CF fog_connection
   bosh -d cf manifest | grep aws_access_key_id
   ```

### 10. Performance Issues

**Symptom:**
- Slow upload/download
- High latency
- Timeouts

**Diagnosis:**

1. **Check disk I/O**
   ```bash
   iostat -x 1
   ```

2. **Check SeaweedFS metrics**
   ```bash
   curl http://localhost:9333/stats/disk
   curl http://localhost:9333/stats/counter
   ```

3. **Increase resources**
   ```yaml
   vm_type: large  # More CPU/RAM
   persistent_disk_type: fast-100GB  # SSD instead of HDD
   ```

## Debug Mode

Enable verbose logging:

```yaml
properties:
  seaweedfs:
    log_level: 0  # 0=debug, 1=info, 2=warn, 3=error
```

Redeploy and check logs for detailed output.

## Getting Additional Help

1. Check SeaweedFS documentation: https://github.com/seaweedfs/seaweedfs/wiki
2. Review BOSH release logs: `bosh -d seaweedfs logs`
3. Check BOSH events: `bosh -d seaweedfs events`
4. Review implementation plan: [IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md)

## Known Limitations (POC)

- Single-node only (no HA)
- Limited production testing
- Basic S3 API coverage
- No advanced SeaweedFS features (replication, erasure coding, etc.)
- No metrics/monitoring integration
- No log forwarding configuration

Remember: **This is a proof-of-concept.** For production use, consider commercial S3-compatible solutions with proper support and SLAs.
