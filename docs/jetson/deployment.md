# Deployment


## Development

### Backend


    ```bash
    cd /idata/intellicare_uat/backend
    uv --project backend run uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8005 --reload --reload-dir app --reload-exclude "backend/.venv/*"
    ```

## Production

### Backend

Edit env variables to jetson in `backend/.env`:
```bash
DEPLOYMENT_MODE=jetson
```

Edit `config.py`:
```bash
DEPLOYMENT_MODE: Literal["laptop", "jetson", "server"] = "jetson"
```

```bash
cd /idata/intellicare_uat/backend
uv run uvicorn app.main:app --host 0.0.0.0 --port 8005
```