# Trivy

Trivy compares packages in the image with its vulnerability database. Run from the repository root with access to the image registry and database. The command extracts the image from the rendered chart, so the scan follows changes to the chart defaults.

```bash
IMAGE="$(helm template webapp ./helm/webapp --show-only templates/deployment.yaml | awk -F '"' '/^[[:space:]]+image: "/ {print $2}')"
test -n "$IMAGE" && trivy image --scanners vuln --severity HIGH,CRITICAL "$IMAGE"
```

Read the CVE ID, package, installed version, severity and fixed version when available. The report can change as the database is updated.

## Optional CI gate

```bash
test -n "$IMAGE" && trivy image --scanners vuln --severity HIGH,CRITICAL --exit-code 1 "$IMAGE"
```

With `--exit-code 1`, findings at the selected severities cause a nonzero exit. Read the output to distinguish findings from scan errors. Then continue with the [Helm lesson](../helm/README.md).

[Trivy exit-code documentation](https://trivy.dev/docs/latest/configuration/others/)
