# QAless — binary releases

Compiled binaries for [QAless](https://github.com/rwrrioe/qaless).

This repo only hosts release artifacts. The source lives at [rwrrioe/qaless](https://github.com/rwrrioe/qaless) (private).

## Install

### macOS (Apple Silicon)
```bash
curl -L https://github.com/rwrrioe/qaless-releases/releases/latest/download/qaless_<version>_darwin_arm64.tar.gz | sudo tar xz -C /usr/local/bin
```

### Windows 11
Download the latest `qaless_*_windows_amd64.zip` from [Releases](https://github.com/rwrrioe/qaless-releases/releases/latest), extract `qaless.exe`, place it on your PATH.

## Run

QAless requires a $5/month license key from [polar.sh/qaless](https://polar.sh/qaless).

```bash
qaless config set ANTHROPIC_API_KEY=sk-ant-...
qaless config set LICENSE_KEY=qal_...
qaless analyze --input "Users can reset password via email link"
```

See the [main repo](https://github.com/rwrrioe/qaless) for full docs.
