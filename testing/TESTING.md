# Testing

## Backend (tradermate)

```bash
cd tradermate
pip install -r requirements.txt
pytest -q
```

## Frontend (tradermate-portal)

```bash
cd tradermate-portal
npm install
npm run test:run
```

## E2E

See `development/frontend/E2E_README.md`.

## Notes

- Some tests require MySQL/Redis to be running (`docker-compose.dev.yml`).
- For a quick smoke test, run `pytest -q tests/` in the backend repo.
