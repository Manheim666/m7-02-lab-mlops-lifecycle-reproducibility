# ETA Model — MLOps Lifecycle

```mermaid
flowchart TD
    DATA[(Data Lake\nraw delivery events\nParquet on S3)]
    EXP[Experimentation\nMLflow Tracking Server]
    TRAIN[Training Pipeline\nweekly cron job]
    EVAL[Evaluation\nfrozen held-out set\nautomated gate check]
    STAGING[Registry: Staging]
    PROD[Registry: Production]
    ARCH[Registry: Archived]
    DEPLOY[Serving\nmodel server · 200 RPS\np95 ≤ 150 ms SLA]
    MON[Monitoring\nEvidentlyAI]

    DATA -->|"dataset_hash — SHA256 of Parquet snapshot\n(DVC-versioned)"| EXP
    EXP -->|"run_id, hyperparams, git_sha\n(logged to MLflow)"| TRAIN
    TRAIN -->|"model.onnx + run_id + eval_metrics"| EVAL
    EVAL -->|"model_uri · AUTO if gates pass\n(MAE ≤ 4.2 min, p95 error ≤ 9.0 min)"| STAGING
    STAGING -->|"canary_report + approval_token\nMANUAL: ML lead + Ops"| PROD
    PROD -->|"model_uri, version_tag\n(on supersession)"| ARCH
    PROD -->|"model_uri, version_tag"| DEPLOY
    DEPLOY -->|"inference_logs\n(score, latency_p95, request_count)"| MON
    MON -->|"drift_signal\n(PSI > 0.2 on input features\nor MAE degradation > 10%)"| TRAIN
    MON -->|"latency_alert\n(p95 breaches SLA)"| DEPLOY
```

## Transition annotations

| Arrow | Automatic or Manual | Trigger |
|---|---|---|
| Evaluation → Staging | **Automatic** | All gates pass (MAE + p95 error thresholds on frozen eval set) |
| Staging → Production | **Manual** | ML lead + Ops approve after reviewing canary report |
| Production → Archived | **Automatic** | Triggered when a new model is promoted to Production |
| Monitoring → Training | **Automatic** | Drift signal emitted when PSI > 0.2 on any input feature or MAE degrades > 10% vs. baseline |
| Monitoring → Serving | **Automatic** | Latency alert if p95 breaches 150 ms SLA for > 5 min |
