# Octra Chain → BigQuery + Stats Dashboard

Indexes the Octra blockchain into Google BigQuery and publishes a pre-aggregated
stats JSON to Google Cloud Storage.

## GCP Resources

| Resource | Name |
|---|---|
| Project | `octra-data` |
| BigQuery dataset | `devnet` |
| GCS bucket | `octra-data` (public read) |
| Stats JSON | `gs://octra-data/devnet/stats.json` |
| Public URL | `https://storage.googleapis.com/octra-data/devnet/stats.json` |
