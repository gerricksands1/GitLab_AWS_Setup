# Lessons Learned

## Errors

- GitLab requires a lot of resources. At minimum you need at least 2 cores and 8 gigabytes of ram for a fairly workable GUI experience.
- What can be missed is storage. It's best to have at least 20 GB of storage for this to work well without regular issues.
- If storage is minimal say around 8 GB the meta data will quickly fill up after a few days causing 500, 504, and 503 errors.
- These errors can cause your site to not work, cause issues with pushing and pulling via Git, and causes issues accessing Linux through Session Manager.
- One of the solutions is to turn off "Prometheus_Monitoring" for memory purposes and then adding more storage via LVM for the other errors.