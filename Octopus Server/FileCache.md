# File Cache

## Cleanup

The clean file cache [job](Job.md) runs every 4 hours.

The job cleans up packages and delta compression files if one of the following criteria is met
- The file was created more than `DaysToCachePackages` ago and has not been accessed in the last 3 hours
- There is low disk space `LowDiskSpaceThresholdGigabytes` (taking into account planned deletions
- The cache directory limit has been exceeded `CacheDirectoryFullThresholdGigabytes` (taking into account planned deletions)
