# SSL Certificate Verification Issue for Users Behind Corporate Proxies

## Problem

When running examples (both Jupyter notebooks and Marimo notebooks) that download data over HTTPS, users behind corporate networks with SSL/TLS inspection (man-in-the-middle proxies) encounter SSL certificate verification errors.

### Error Example

```
SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self-signed certificate in certificate chain (_ssl.c:1028)
```

This occurs when:
- Running `uvx juv run examples/air-traffic-control.ipynb`
- Running `uv run marimo edit examples/marimo/nyc_taxi_trips.py --sandbox`
- Any code that uses `requests.get()` to download files from GitHub or other HTTPS sources

## Root Cause

The corporate proxy intercepts HTTPS connections and presents its own certificate chain, which includes a self-signed corporate CA certificate. Python's `requests` library (via `urllib3`) doesn't trust this certificate by default because it's not in the system's certificate store.

## Solution

Users need to configure Python/requests to use a certificate bundle that includes the corporate CA certificate. The fix involves:

1. **Adding the corporate CA certificate to a custom certificate bundle** (e.g., `~/.certs/ca-bundle.crt`)
2. **Modifying code to use this bundle** by passing it to `requests.get(verify=cert_file)`

### Implementation

For notebooks/examples that use `requests.get()`, modify the code to:

```python
import os

# Get certificate bundle location
cert_file = os.getenv("SSL_CERT_FILE", os.path.expanduser("~/.certs/ca-bundle.crt"))
verify = cert_file if os.path.exists(cert_file) else True

# Use in requests call
r = requests.get(url, verify=verify)
```

This approach:
- Checks the `SSL_CERT_FILE` environment variable first (standard Python/OpenSSL convention)
- Falls back to `~/.certs/ca-bundle.crt` if not set
- Falls back to default system certificates if the custom bundle doesn't exist
- Works in both Jupyter notebooks and Marimo notebooks

## Suggested Documentation Addition

It would be helpful to add a section to the repository documentation (e.g., in `README.md` or `DEVELOP.md`) covering:

1. **For users behind corporate proxies:**
   - How to obtain their corporate CA certificate
   - How to add it to a certificate bundle: `~/.certs/ca-bundle.crt`
   - How to set the `SSL_CERT_FILE` environment variable
   - That examples may need to be modified to use custom certificates (or ideally, examples should already support this)

2. **For maintainers:**
   - Consider updating all examples that use `requests.get()` to support custom certificate bundles
   - This would make the examples work out-of-the-box for users behind corporate proxies

## Related Files

- `examples/air-traffic-control.ipynb` - Jupyter notebook with `requests.get()` call
- `examples/marimo/nyc_taxi_trips.py` - Marimo notebook with `requests.get()` call

Both of these files were updated to support custom certificate bundles to resolve this issue.
