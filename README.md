# Android MultiBranch Pipeline — BasicApp

Pipeline files for the `VirgoAndroidMultiBranch_ARM64` CloudBees CI training example.
See [`android_pipeline_readme.md`](../android_pipeline_readme.md) for full setup instructions.

| File | Purpose |
|---|---|
| `Jenkinsfile` | Declarative Pipeline — mirrors VirgoAndroidMultiBranch_ARM64 stage structure |
| `multibranch-config.xml` | CloudBees CI MultiBranch `config.xml` — import via CasC or Job DSL |
| `app/BasicApp/` | Minimal Android app used as the pipeline build target |
