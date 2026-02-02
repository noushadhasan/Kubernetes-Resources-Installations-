# Install Kubectl reach plugin 
## Run the following command in terminal 
*Step 1: Install krew (one-time setup)
```bash
(
  set -x
  cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-${OS}_${ARCH}.tar.gz" &&
  tar zxvf krew-${OS}_${ARCH}.tar.gz &&
  ./krew-${OS}_${ARCH} install krew
)
```
*Step 2: Add krew to PATH (important!)
```bash
    export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```
* To make it permanent 👇
```bash
    echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    kubectl krew version
```
* Step 3: Install reach
```bash
    kubectl krew install reach
```
* Check the version
```bash
    kubectl reach --help
```
```bash
kubectl reach people-desk-arl-api-5b7f9cb96b-ldpcb --to www.google.com:443
```


