# Agent Guide for x509-certificate-exporter

## Repository Overview

This repository contains deployment configuration for the x509-certificate-exporter, a Prometheus exporter that monitors X.509 certificates stored in Kubernetes secrets and exposes metrics about their expiration status.

**Purpose**: Deploy and configure x509-certificate-exporter on OpenShift/Kubernetes clusters to monitor TLS certificate expiration.

## Key Components

### OpenShift Template (`openshift/template.yml`)

The template defines the complete deployment stack:

1. **ServiceAccount**: `x509-certificate-exporter` - Identity for the exporter pods
2. **Role/RoleBinding**: RBAC permissions to read secrets in the namespace
3. **Secret**: Service account token for authentication
4. **Deployment**: Runs the exporter with 3 replicas (configurable)
5. **Service**: Exposes metrics endpoint on port 8000

### Exporter Configuration

The exporter runs with these key arguments:
- `--expose-relative-metrics`: Expose time-relative metrics
- `--watch-kube-secrets`: Monitor Kubernetes secrets
- `--secret-type`: Type of secrets to watch (default: `kubernetes.io/tls:tls.crt`)
- `--include-namespace`: Only watch secrets in the pod's namespace
- `--expose-per-cert-error-metrics`: Expose detailed error metrics
- `--listen-address`: Metrics endpoint (default: `:8000`)

## Template Parameters

Key configurable parameters:

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `IMAGE` | `quay.io/app-sre/x509-certificate-exporter` | Container image |
| `IMAGE_VERSION` | `latest` | Image version/tag |
| `X509_CE_PORT` | `8000` | Metrics port |
| `X509_CE_REPLICAS` | `3` | Number of pod replicas |
| `X509_CE_MEMORY_REQUESTS` | `64Mi` | Memory request per pod |
| `X509_CE_MEMORY_LIMITS` | `64Mi` | Memory limit per pod |
| `X509_CE_CPU_REQUESTS` | `10m` | CPU request per pod |
| `SECRET_TYPE` | `kubernetes.io/tls:tls.crt` | Secret type and key to watch |

## Common Tasks

### Modifying Resource Limits

When adjusting memory or CPU:
1. Update corresponding parameters in the template
2. Consider the number of certificates being monitored
3. Ensure requests match limits for guaranteed QoS (memory) or allow burst (CPU)

### Changing the Number of Replicas

The deployment uses pod anti-affinity to spread replicas across nodes:
- Pods are scheduled on different hosts via `requiredDuringSchedulingIgnoredDuringExecution`
- Ensure the target cluster has enough nodes for the replica count

### Adding Custom Annotations

Use the custom annotation parameters for deployment-level annotations:
- `X509_CE_CUSTOM_DEPLOYMENT_ANNOTATION_01`: Annotation key
- `X509_CE_CUSTOM_DEPLOYMENT_ANNOTATION_VAL_01`: Annotation value

Example use case: Disabling specific DVO (Deployment Validation Operator) checks

### Monitoring Different Secret Types

To watch different secret types, modify the `SECRET_TYPE` parameter:
- Format: `<secret-type>:<key-name>`
- Can specify multiple types (comma-separated or multiple `--secret-type` flags)

## Important Considerations

### RBAC Permissions

The Role only grants `get`, `watch`, and `list` on secrets in the namespace. When modifying:
- Keep permissions minimal (principle of least privilege)
- The exporter does NOT need write permissions
- Namespace is isolated via `--include-namespace` flag

### High Availability

The deployment is configured for HA:
- 3 replicas by default
- Pod anti-affinity ensures distribution
- `preStop` hook with 5-second sleep for graceful shutdown
- Separate readiness and liveness probes

### Health Checks

Both probes use the `/metrics` endpoint:
- **Readiness**: Initial delay 10s, period 15s
- **Liveness**: Initial delay 15s, period 15s

If metrics endpoint fails, the pod will be restarted.

### Resource Management

- No CPU limits set (annotation: `ignore-check.kube-linter.io/unset-cpu-requirements`)
- Memory is strictly limited to prevent OOM issues
- Adjust based on actual certificate count in the cluster

### Common Issues

- **Pods not spreading**: Cluster may not have enough nodes for anti-affinity
- **No metrics**: Check RBAC permissions and secret types match
- **High memory usage**: Increase limits based on certificate count
- **Image pull errors**: Verify `IMAGE_PULL_SECRET` is configured correctly

## Making Changes

### When Modifying the Template

1. **Parameter changes**: Update default values and documentation
2. **Container args**: Ensure flags are compatible with the image version
3. **RBAC changes**: Minimize permissions, document why changes are needed
4. **Resource changes**: Consider impact on cluster capacity and certificate count
5. **Label changes**: Maintain consistency across all resources

### Version Updates

When updating `IMAGE_VERSION`:
1. Review changelog of the upstream exporter
2. Check for new command-line flags or deprecated options
3. Test in a non-production environment first
4. Consider resource requirement changes

## Upstream Project

The x509-certificate-exporter is an external project. This repository only contains deployment configuration.

- Upstream: [enix/x509-certificate-exporter](https://github.com/enix/x509-certificate-exporter)
- Documentation: Check upstream repository for exporter functionality
- Issues: File deployment-specific issues here; exporter issues upstream

## Integration Notes

### Prometheus Integration

The service exposes metrics on the `metrics` port. ServiceMonitor or PodMonitor resources may be needed depending on your Prometheus operator configuration.

### App-SRE Context

This repository appears to be part of an App-SRE (Application Site Reliability Engineering) setup:
- Image sourced from `quay.io/app-sre`
- Deployment patterns follow App-SRE conventions
- Consider integration with other app-sre tooling and pipelines
